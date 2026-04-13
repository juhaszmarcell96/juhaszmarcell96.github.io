# Print In Order

```cpp
// =============================================================================
//
// Three functions run on three separate threads. We must guarantee they
// execute in the order: first -> second -> third.
//
// Approach A (promise/future): Two one-shot signals. Thread 1 sets
// promise_1 after finishing; thread 2 waits on future_1 then sets
// promise_2; thread 3 waits on future_2.
//
// Approach B (condition_variable + flag): A shared counter tracks which
// step we're on. Each thread waits until the counter reaches its turn.
//
// Both are shown below.
// =============================================================================

// --- Approach A: promise / future ---

class PrintInOrderFuture {
public:
    PrintInOrderFuture()
        : m_future_1{m_promise_1.get_future()}
        , m_future_2{m_promise_2.get_future()}
    {
    }

    void first(std::function<void()> print_first)
    {
        print_first();
        m_promise_1.set_value();  // signal thread 2
    }

    void second(std::function<void()> print_second)
    {
        m_future_1.wait();        // wait for thread 1
        print_second();
        m_promise_2.set_value();  // signal thread 3
    }

    void third(std::function<void()> print_third)
    {
        m_future_2.wait();        // wait for thread 2
        print_third();
    }

private:
    std::promise<void> m_promise_1;
    std::promise<void> m_promise_2;
    std::shared_future<void> m_future_1;
    std::shared_future<void> m_future_2;
};

// --- Approach B: condition_variable + step counter ---

class PrintInOrderCV {
public:
    void first(std::function<void()> print_first)
    {
        print_first();
        {
            std::lock_guard<std::mutex> lock{m_mutex};
            m_step = 2;
        }
        m_cv.notify_all();
    }

    void second(std::function<void()> print_second)
    {
        {
            std::unique_lock<std::mutex> lock{m_mutex};
            // Must use a while-loop (or predicate overload) because
            // condition_variable can have spurious wakeups.
            m_cv.wait(lock, [this]() { return m_step >= 2; });
        }
        print_second();
        {
            std::lock_guard<std::mutex> lock{m_mutex};
            m_step = 3;
        }
        m_cv.notify_all();
    }

    void third(std::function<void()> print_third)
    {
        {
            std::unique_lock<std::mutex> lock{m_mutex};
            m_cv.wait(lock, [this]() { return m_step >= 3; });
        }
        print_third();
    }

private:
    std::mutex m_mutex;
    std::condition_variable m_cv;
    int m_step{1};
};

// =============================================================================
// Tests
// =============================================================================

void test_print_in_order_future()
{
    std::cout << "--- Print In Order (promise/future) ---\n";

    std::string output;
    std::mutex output_mutex;

    PrintInOrderFuture printer;

    // Launch threads in REVERSE order to prove synchronization works.
    std::thread t3{[&]() {
        printer.third([&]() {
            std::lock_guard<std::mutex> lock{output_mutex};
            output += "third";
        });
    }};

    std::thread t2{[&]() {
        printer.second([&]() {
            std::lock_guard<std::mutex> lock{output_mutex};
            output += "second";
        });
    }};

    std::thread t1{[&]() {
        printer.first([&]() {
            std::lock_guard<std::mutex> lock{output_mutex};
            output += "first";
        });
    }};

    t1.join();
    t2.join();
    t3.join();

    assert(output == "firstsecondthird");
    std::cout << "  Output: " << output << "\n";
    std::cout << "  Passed.\n\n";
}

void test_print_in_order_cv()
{
    std::cout << "--- Print In Order (condition_variable) ---\n";

    std::string output;
    std::mutex output_mutex;

    PrintInOrderCV printer;

    // Again, launch in reverse order.
    std::thread t3{[&]() {
        printer.third([&]() {
            std::lock_guard<std::mutex> lock{output_mutex};
            output += "third";
        });
    }};

    std::thread t1{[&]() {
        printer.first([&]() {
            std::lock_guard<std::mutex> lock{output_mutex};
            output += "first";
        });
    }};

    std::thread t2{[&]() {
        printer.second([&]() {
            std::lock_guard<std::mutex> lock{output_mutex};
            output += "second";
        });
    }};

    t1.join();
    t2.join();
    t3.join();

    assert(output == "firstsecondthird");
    std::cout << "  Output: " << output << "\n";
    std::cout << "  Passed.\n\n";
}
```