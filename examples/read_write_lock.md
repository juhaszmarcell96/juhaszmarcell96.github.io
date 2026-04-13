# Read-Write Lock

```cpp
// =============================================================================
//
// Multiple readers can hold the lock simultaneously, but a writer needs
// exclusive access. This implementation gives PRIORITY TO WRITERS to
// prevent writer starvation:
//
//   - A new reader blocks if any writer is waiting or active.
//   - A writer blocks until all active readers finish.
//
// Without writer priority, a steady stream of readers can starve
// writers indefinitely. The trick is the m_writers_waiting counter:
// when it is > 0, new readers must wait.
//
// State transitions:
//   Reader wants in:  wait while (m_active_writers > 0 || m_writers_waiting > 0)
//   Reader done:      decrement m_active_readers, notify if it hits 0
//   Writer wants in:  increment m_writers_waiting, wait while (m_active_readers > 0 || m_active_writers > 0)
//   Writer done:      decrement m_active_writers, notify all
// =============================================================================

class ReadWriteLock {
public:
    void read_lock()
    {
        std::unique_lock<std::mutex> lock{m_mutex};
        // Block if a writer is active OR waiting (writer priority).
        m_cv.wait(lock, [this]() {
            return m_active_writers == 0 && m_writers_waiting == 0;
        });
        ++m_active_readers;
    }

    void read_unlock()
    {
        std::lock_guard<std::mutex> lock{m_mutex};
        --m_active_readers;
        if (m_active_readers == 0) {
            // Last reader out -- wake up a waiting writer (if any).
            m_cv.notify_all();
        }
    }

    void write_lock()
    {
        std::unique_lock<std::mutex> lock{m_mutex};
        ++m_writers_waiting;
        // Block until no readers and no other writer is active.
        m_cv.wait(lock, [this]() {
            return m_active_readers == 0 && m_active_writers == 0;
        });
        --m_writers_waiting;
        ++m_active_writers;
    }

    void write_unlock()
    {
        std::lock_guard<std::mutex> lock{m_mutex};
        --m_active_writers;
        // Wake all -- both readers and writers check their conditions.
        m_cv.notify_all();
    }

private:
    std::mutex m_mutex;
    std::condition_variable m_cv;
    int m_active_readers{0};
    int m_active_writers{0};
    int m_writers_waiting{0};  // key to writer priority
};

// RAII wrappers for clean usage (and to avoid forgetting unlock).

class ReadGuard {
public:
    explicit ReadGuard(ReadWriteLock& rw_lock)
        : m_lock{rw_lock}
    {
        m_lock.read_lock();
    }

    ~ReadGuard()
    {
        m_lock.read_unlock();
    }

    ReadGuard(const ReadGuard&) = delete;
    ReadGuard& operator=(const ReadGuard&) = delete;

private:
    ReadWriteLock& m_lock;
};

class WriteGuard {
public:
    explicit WriteGuard(ReadWriteLock& rw_lock)
        : m_lock{rw_lock}
    {
        m_lock.write_lock();
    }

    ~WriteGuard()
    {
        m_lock.write_unlock();
    }

    WriteGuard(const WriteGuard&) = delete;
    WriteGuard& operator=(const WriteGuard&) = delete;

private:
    ReadWriteLock& m_lock;
};

// =============================================================================
// Tests
// =============================================================================

void test_read_write_lock()
{
    std::cout << "--- Read-Write Lock ---\n";

    ReadWriteLock rw_lock;
    int shared_data = 0;
    std::atomic<int> read_count{0};
    std::atomic<int> write_count{0};

    constexpr int num_readers = 8;
    constexpr int num_writers = 3;
    constexpr int ops_per_thread = 200;

    // Readers: read shared_data many times, verify it's consistent
    // (always a multiple of 10, since writers always write multiples of 10).
    auto reader_task = [&]() {
        for (int i = 0; i < ops_per_thread; ++i) {
            ReadGuard guard{rw_lock};
            int val = shared_data;
            // If the lock works, we should never see a partial write.
            assert(val % 10 == 0);
            ++read_count;
        }
    };

    // Writers: increment shared_data by 10 (non-atomically, in two steps,
    // to expose any concurrency bugs).
    auto writer_task = [&]() {
        for (int i = 0; i < ops_per_thread; ++i) {
            WriteGuard guard{rw_lock};
            // Two-step write to make races detectable.
            shared_data += 7;
            // A reader seeing this intermediate state would fail the
            // (val % 10 == 0) assertion.
            std::this_thread::yield();
            shared_data += 3;
            ++write_count;
        }
    };

    std::vector<std::thread> threads;
    for (int i = 0; i < num_readers; ++i) {
        threads.emplace_back(reader_task);
    }
    for (int i = 0; i < num_writers; ++i) {
        threads.emplace_back(writer_task);
    }
    for (auto& t : threads) {
        t.join();
    }

    int expected_data = num_writers * ops_per_thread * 10;
    assert(shared_data == expected_data);
    std::cout << "  Readers completed: " << read_count << "\n";
    std::cout << "  Writers completed: " << write_count << "\n";
    std::cout << "  Final shared_data: " << shared_data
              << " (expected " << expected_data << ")\n";
    std::cout << "  No torn reads detected. Passed.\n\n";
}
```