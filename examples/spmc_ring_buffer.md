```cpp
template<typename T, std::size_t N>
requires std::is_trivially_copyable_v<T> &&
         std::is_standard_layout_v<T> &&
         std::is_trivially_destructible_v<T>
class LockFreeRingBuffer {
    static_assert((N > 0) && ((N & (N - 1)) == 0), "Buffer size must be a power of 2");
private:
    struct Slot {
        alignas(CACHE_LINE_SIZE) std::atomic<uint64_t> sequence;
        T data;
    };
    alignas(CACHE_LINE_SIZE) std::unique_ptr<Slot[]> m_slots;

    alignas(CACHE_LINE_SIZE) std::atomic<uint64_t> m_read_position;
    alignas(CACHE_LINE_SIZE) std::atomic<uint64_t> m_write_position;
    static_assert(decltype(m_write_position)::is_always_lock_free, "write position must be lock-free");
    static_assert(decltype(m_read_position)::is_always_lock_free, "read position must be lock-free");
public:
    LockFreeRingBuffer() : m_slots(std::make_unique<Slot[]>(N)), m_read_position(0), m_write_position(0) {
        // initialize sequences: slot i has sequence i
        for (size_t i = 0; i < N; ++i) {
            m_slots[i].sequence.store(i, std::memory_order_relaxed);
        }
    }
    // deleted constructors
    LockFreeRingBuffer(const LockFreeRingBuffer&) = delete;            // no copy constructor
    LockFreeRingBuffer(LockFreeRingBuffer&&) = delete;                 // no move constructor
    LockFreeRingBuffer& operator=(const LockFreeRingBuffer&) = delete; // no copy assignment
    LockFreeRingBuffer& operator=(LockFreeRingBuffer&&) = delete;      // no move assignment

    template<typename U>
    requires std::is_same_v<std::decay_t<U>, T>
    bool try_write(U&& value) noexcept {
        std::uint64_t write_pos = m_write_position.load(std::memory_order_relaxed);
        Slot& slot = m_slots[write_pos & (N - 1)];
        std::uint64_t seq = slot.sequence.load(std::memory_order_acquire);

        // check if slot is available (sequence should match write position)
        if (seq != write_pos) {
            return false; // slot not yet consumed
        }
        
        // write data
        slot.data = std::forward<U>(value);
        
        // mark as ready for consumers (increment sequence)
        slot.sequence.store(write_pos + 1, std::memory_order_release);
    
        // update write position (only we touch this)
        m_write_position.store(write_pos + 1, std::memory_order_relaxed);
        
        return true;
    }
    
    std::optional<T> try_read() noexcept {
        std::uint64_t read_pos = m_read_position.load(std::memory_order_relaxed);
        Slot& slot = m_slots[read_pos & (N - 1)];
        std::uint64_t seq = slot.sequence.load(std::memory_order_acquire);
        
        // check if data is ready (sequence should be read_pos + 1)
        if (seq != read_pos + 1) {
            return std::nullopt; // Not yet written or already consumed
        }
        
        // try to claim this read position
        if (!m_read_position.compare_exchange_weak(
            read_pos, read_pos + 1,
            std::memory_order_acquire,
            std::memory_order_relaxed)) {
            return std::nullopt;
        }
        
        // read data (we know it's valid because sequence matched)
        T value = slot.data;
        
        // mark slot as consumed (advance sequence by N for next cycle)
        slot.sequence.store(read_pos + N, std::memory_order_release);
        
        return value;
    }
    
    // returns the number of elements available to read
    // WARNING: this function is racy, it can return size from the past
    // between reading the write_pos and the read_pos, another thread could have modified them
    std::size_t size() const noexcept {
        std::uint64_t write_pos = m_write_position.load(std::memory_order_acquire);
        std::uint64_t read_pos = m_read_position.load(std::memory_order_acquire);
        return write_pos > read_pos ? write_pos - read_pos : 0;
    }
    
    // returns approximate free slots (can be inaccurate due to races)
    std::size_t available() const noexcept {
        return N - size();
    }
    
    // check if the buffer is empty
    bool empty() const noexcept {
        return size() == 0;
    }
    
    // check if the buffer is full
    bool full() const noexcept {
        return size() == N;
    }
};
```