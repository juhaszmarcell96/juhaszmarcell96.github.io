```cpp
template<typename T, std::size_t N>
requires std::is_trivially_copyable_v<T> &&
         std::is_trivially_destructible_v<T>
class LockFreeRingBuffer {
    static_assert((N > 0) && ((N & (N - 1)) == 0), "Buffer size must be a power of 2");
public:
    using value_type = T;
private:
    // sequence number meanings:
    // - seq == pos     : Cell is ready for writing at position pos
    // - seq == pos + 1 : Cell contains data written at pos (ready for reading)
    // - seq == pos + N : Cell was read and is ready for next writing cycle
    struct Cell {
        alignas(CACHE_LINE_SIZE) std::atomic<std::uint64_t> sequence;
        T data;
    };
    
    alignas(CACHE_LINE_SIZE) std::unique_ptr<Cell[]> m_buffer;
    alignas(CACHE_LINE_SIZE) std::atomic<std::uint64_t> m_write_position;
    alignas(CACHE_LINE_SIZE) std::atomic<std::uint64_t> m_read_position;
    
    static_assert(decltype(m_write_position)::is_always_lock_free, "write position must be lock-free");
    static_assert(decltype(m_read_position)::is_always_lock_free, "read position must be lock-free");
    
    // helper functions for improved readability
    static constexpr std::size_t wrap(std::uint64_t pos) noexcept { return static_cast<std::size_t>(pos & (N - 1)); }
    static constexpr std::uint64_t seq_for_write(std::uint64_t pos) noexcept { return pos; }
    static constexpr std::uint64_t seq_for_read(std::uint64_t pos) noexcept { return pos + 1; }
    static constexpr std::uint64_t seq_for_next_cycle(std::uint64_t pos) noexcept { return pos + N; }

public:
    LockFreeRingBuffer() : m_buffer(std::make_unique<Cell[]>(N)), m_write_position(0), m_read_position(0) {
        // initialize sequences: cell i is ready for writing at position i
        for (std::size_t i = 0; i < N; ++i) {
            m_buffer[i].sequence.store(seq_for_write(i), std::memory_order_relaxed);
        }
    }
    
    // Deleted constructors
    LockFreeRingBuffer(const LockFreeRingBuffer&) = delete;
    LockFreeRingBuffer(LockFreeRingBuffer&&) = delete;
    LockFreeRingBuffer& operator=(const LockFreeRingBuffer&) = delete;
    LockFreeRingBuffer& operator=(LockFreeRingBuffer&&) = delete;

    // try to write an element to the buffer
    // returns true if successful, false if buffer is full
    // thread-safe for multiple producers
    template<typename U>
    requires std::is_same_v<std::decay_t<U>, T>
    bool try_write(U&& value) noexcept {
        // load current write position
        // relaxed ordering: we only care about claiming a position via CAS
        // the sequence numbers provide the actual synchronization for data
        std::uint64_t pos = m_write_position.load(std::memory_order_relaxed);
        
        while (true) {
            Cell& cell = m_buffer[wrap(pos)];
            
            // acquire: ensure we see previous writes to this cell
            std::uint64_t seq = cell.sequence.load(std::memory_order_acquire);
            
            // calculate difference to determine cell state
            // diff == 0  : cell ready for writing at this position
            // diff < 0   : cell not yet consumed (buffer full)
            // diff > 0   : position already claimed by another thread (stale pos)
            std::int64_t diff = static_cast<std::int64_t>(seq) - static_cast<std::int64_t>(seq_for_write(pos));
            
            if (diff == 0) {
                // cell is available, try to claim this position
                if (m_write_position.compare_exchange_weak(
                    pos, pos + 1,
                    std::memory_order_relaxed,  // success: just claiming position
                    std::memory_order_relaxed)) { // failure: pos updated, retry
                    
                    // successfully claimed position, write data
                    cell.data = std::forward<U>(value);
                    
                    // release: make data write visible before marking as readable
                    cell.sequence.store(seq_for_read(pos), std::memory_order_release);
                    
                    return true;
                }
                // CAS failed, pos was updated, loop will retry with new pos
            }
            else if (diff < 0) {
                // buffer is full (cell not yet consumed)
                return false;
            }
            else {
                // position is stale (another thread already claimed it)
                // reload current position and retry
                pos = m_write_position.load(std::memory_order_relaxed);
            }
        }
    }
    
    // try to read an element from the buffer
    // returns the value if successful, nullopt if buffer is empty
    // thread-safe for multiple consumers
    std::optional<T> try_read() noexcept {
        // load current dequeue position
        std::uint64_t pos = m_read_position.load(std::memory_order_relaxed);
        
        while (true) {
            Cell& cell = m_buffer[wrap(pos)];
            
            // acquire: ensure we see the data written by producer
            std::uint64_t seq = cell.sequence.load(std::memory_order_acquire);
            
            // calculate difference to determine cell state
            // diff == 0  : cell ready for reading
            // diff < 0   : cell not yet written (buffer empty)
            // diff > 0   : position already consumed by another thread (stale pos)
            std::int64_t diff = static_cast<std::int64_t>(seq) - static_cast<std::int64_t>(seq_for_read(pos));
            
            if (diff == 0) {
                // cell has data, try to claim this position
                if (m_read_position.compare_exchange_weak(
                    pos, pos + 1,
                    std::memory_order_relaxed,  // success: just claiming position
                    std::memory_order_relaxed)) { // failure: pos updated, retry
                    
                    // successfully claimed position, read data
                    T value = cell.data;
                    
                    // release: mark cell as consumed and ready for next cycle
                    cell.sequence.store(seq_for_next_cycle(pos), std::memory_order_release);
                    
                    return value;
                }
                // CAS failed, pos was updated, loop will retry with new pos
            }
            else if (diff < 0) {
                // buffer is empty (cell not yet written)
                return std::nullopt;
            }
            else {
                // position is stale (another thread already consumed it)
                // reload current position and retry
                pos = m_read_position.load(std::memory_order_relaxed);
            }
        }
    }
    
    // returns approximate number of elements in buffer
    // WARNING: This is racy and only suitable for monitoring/metrics
    // DO NOT use for correctness - always check try_read()/try_write() return values
    std::size_t size() const noexcept {
        // read positions - these can change between the two loads
        std::uint64_t enqueue = m_write_position.load(std::memory_order_acquire);
        std::uint64_t dequeue = m_read_position.load(std::memory_order_acquire);
        
        // handle wrap-around safely
        if (enqueue >= dequeue) {
            std::uint64_t diff = enqueue - dequeue;
            return diff <= N ? static_cast<std::size_t>(diff) : N;
        }
        return 0;
    }
    
    // returns approximate number of free slots (racy, for monitoring only)
    std::size_t available() const noexcept {
        std::size_t current_size = size();
        return current_size < N ? N - current_size : 0;
    }
    
    // returns capacity
    std::size_t capacity() const noexcept {
        return N;
    }
    
    // check if buffer appears empty (racy)
    bool empty() const noexcept {
        return size() == 0;
    }
    
    // check if buffer appears full (racy)
    bool full() const noexcept {
        return size() >= N;
    }
};
```