```cpp
// it is possible that some compilers do not implement this or have it wrong -> fall back to 64 (most common)
static constexpr std::size_t CACHE_LINE_SIZE { std::hardware_destructive_interference_size };

template<typename T, std::size_t N>
requires std::is_trivially_copyable_v<T> && std::is_trivially_destructible_v<T>
class LockFreeRingBuffer {
private:
    // Why a power of 2? Because it helps with index wrapping. "index % size" wraps the index around.
    // But modulo is a division operation, which is slow. However, dividing by a power of two is fast,
    // because it is converted to a right bitshift by n, where n is the power (2^n).
    static_assert((N > 0) && ((N & (N - 1)) == 0), "Buffer size must be a power of 2");
public:
    using value_type = T;
private:
    // false sharing : thread A only writes the "write index", but the "read index" is on the same cache line so it is invalidated as well
    // align the data, so that when the producer writes the data it does not
    // invalidate the atomics because of false sharing
    alignas(CACHE_LINE_SIZE) std::array<T, N> m_data;
    // align atomics to cache lines to avoid false sharing
    alignas(CACHE_LINE_SIZE) std::atomic<uint64_t> m_read_position;
    alignas(CACHE_LINE_SIZE) std::atomic<uint64_t> m_write_position;
    static_assert(decltype(m_write_position)::is_always_lock_free, "write position must be lock-free");
    static_assert(decltype(m_read_position)::is_always_lock_free, "read position must be lock-free");
public:
    LockFreeRingBuffer() : m_read_position(0), m_write_position(0) {}
    // deleted constructors
    LockFreeRingBuffer(const LockFreeRingBuffer&) = delete;            // no copy constructor
    LockFreeRingBuffer(LockFreeRingBuffer&&) = delete;                 // no move constructor
    LockFreeRingBuffer& operator=(const LockFreeRingBuffer&) = delete; // no copy assignment
    LockFreeRingBuffer& operator=(LockFreeRingBuffer&&) = delete;      // no move assignment

    // try to write an element to the buffer
    // returns true if successful, false if buffer is full
    // use perfect forwarding to support move semantics as well for performance
    // the std::is_same_v might be too strict, std::is_convertible_v or std::is_constructible_v could be considered
    template<typename U>
    requires std::is_same_v<std::decay_t<U>, T>
    bool try_write(U&& value) noexcept {
        // we are working with the assumption of a single producer thread
        // -> the try_write function cannot be called concurrently
        // this is why the write position can be read with a relaxed memory order, because this is the only thread that modifies it
        // the read position must be read with acquire semantics, which guarantees that all prior writes must be visible to us
        std::uint64_t write_pos = m_write_position.load(std::memory_order_relaxed);
        std::uint64_t read_pos = m_read_position.load(std::memory_order_acquire);
        
        // check if buffer is full
        // the difference between the read and write position cannot go above the buffer size
        // why this works? because the read position and the write position are never reset to 0, they are just incremented into "infinity"
        // should we be worried about overflow? -> not really, it would take many-many years, so it is not a critical issue
        // this strategy simplifies the logic a lot
        if (write_pos - read_pos >= N) {
            return false;
        }
        
        // write the value
        std::size_t index = write_pos & (N - 1); // same as "write_pos % N" -> index wraps around -> much more efficient (don't forget the static assert)
        m_data[index] = std::forward<U>(value);
        
        // ensure the write is visible to readers before updating the write position
        // release ensures:
        //     - writing m_data in this thread is not moved after writing the write position
        //     - the memory fence ensures that the write to m_data become visible to any thread that later does an acquire load on the write position
        m_write_position.store(write_pos + 1, std::memory_order_release);
        
        return true;
    }
    
    // try to read an element from the buffer
    // returns the value if successful, nullopt if buffer is empty
    // if the type would not be is_trivially_copyable, then the optional could throw
    std::optional<T> try_read() noexcept {
        // we are working with a single consumer -> this is the only thread writing the read position -> relaxed memory order
        // reading the write position with acquire is necessary, because it guarantees that we see the modified m_store that the producer thread released
        std::uint64_t read_pos = m_read_position.load(std::memory_order_relaxed);
        std::uint64_t write_pos = m_write_position.load(std::memory_order_acquire);
        
        // check if buffer is empty
        if (read_pos >= write_pos) {
            return std::nullopt;
        }
        
        // read the value
        std::size_t index = read_pos & (N - 1);
        T value = m_data[index];
        
        // update read position
        m_read_position.store(read_pos + 1, std::memory_order_release);
        
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
    
    // returns capacity
    std::size_t capacity() const noexcept {
        return N;
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