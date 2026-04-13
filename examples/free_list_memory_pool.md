# Simple Fixed-Size Memory Pool

Pre-allocate one contiguous block and carve it into fixed-size chunks.
The free list is intrusive: we store the next-pointer INSIDE each free
chunk (since the memory is unused, we can reuse it for bookkeeping).

- **allocate**: pop the head of the free list. O(1)
- **deallocate**: push the chunk back onto the free list. O(1)

Advantages over malloc:
- No per-allocation overhead (malloc headers, fragmentation).
- All chunks are contiguous -> better cache locality.
- Deterministic O(1) allocation (no searching free lists of varying sizes like a general-purpose allocator).

This pattern is common in game engines, networking stacks, and
high-frequency trading systems where allocation latency matters.

```cpp
class MemoryPool {
private:
    // This struct is placed inside each free chunk. Since the chunk
    // is not in use, we can safely reinterpret its first bytes as
    // this node. This is the "intrusive free list" trick.
    struct FreeNode {
        FreeNode* next;
    };

    std::size_t m_chunk_size;
    std::size_t m_chunk_count;
    std::vector<std::uint8_t> m_block;  // the raw backing storage
    FreeNode* m_free_head;              // head of the intrusive free list
public:
    // chunk_size: size of each allocation in bytes (must be >= sizeof(void*))
    // chunk_count: how many chunks to pre-allocate
    MemoryPool(std::size_t chunk_size, std::size_t chunk_count)
        : m_chunk_size{chunk_size}
        , m_chunk_count{chunk_count}
        , m_free_head{nullptr}
    {
        // Ensure each chunk can hold at least a pointer for the free list.
        if (m_chunk_size < sizeof(FreeNode)) {
            m_chunk_size = sizeof(FreeNode);
        }

        // Allocate one contiguous block.
        m_block.resize(m_chunk_size * m_chunk_count);

        // Build the free list by linking all chunks together.
        // We walk from the LAST chunk to the FIRST so that the first
        // allocation returns the lowest address (cache-friendly order).
        m_free_head = nullptr;
        for (std::size_t i = m_chunk_count; i > 0; --i) {
            std::size_t offset = (i - 1) * m_chunk_size;
            auto* node = reinterpret_cast<FreeNode*>(&m_block[offset]);
            node->next = m_free_head;
            m_free_head = node;
        }
    }

    // Non-copyable, non-movable (outstanding pointers would dangle).
    MemoryPool(const MemoryPool&) = delete;
    MemoryPool& operator=(const MemoryPool&) = delete;
    MemoryPool(MemoryPool&&) = delete;
    MemoryPool& operator=(MemoryPool&&) = delete;

    ~MemoryPool() = default;

    // Pop one chunk from the free list. Returns nullptr if pool is exhausted.
    [[nodiscard]] void* allocate() noexcept
    {
        if (m_free_head == nullptr) {
            return nullptr;
        }

        // Pop the head.
        FreeNode* chunk = m_free_head;
        m_free_head = chunk->next;
        return static_cast<void*>(chunk);
    }

    // Push a chunk back onto the free list.
    // The caller is responsible for passing a pointer that was returned
    // by this pool's allocate(). Passing anything else is undefined.
    void deallocate(void* ptr) noexcept
    {
        if (ptr == nullptr) {
            return;
        }

        // Push onto the head of the free list.
        auto* node = static_cast<FreeNode*>(ptr);
        node->next = m_free_head;
        m_free_head = node;
    }

    [[nodiscard]] std::size_t chunk_size() const noexcept
    {
        return m_chunk_size;
    }

    [[nodiscard]] std::size_t chunk_count() const noexcept
    {
        return m_chunk_count;
    }
};

// =============================================================================
// Tests
// =============================================================================

void test_memory_pool()
{
    std::cout << "--- Memory Pool ---\n";

    // Basic allocate/deallocate cycle
    MemoryPool pool{32, 4};

    void* a = pool.allocate();
    void* b = pool.allocate();
    void* c = pool.allocate();
    void* d = pool.allocate();
    assert(a != nullptr && b != nullptr && c != nullptr && d != nullptr);

    // Pool is exhausted
    void* e = pool.allocate();
    assert(e == nullptr);

    // Return one chunk, then allocate again -- should succeed
    pool.deallocate(b);
    void* f = pool.allocate();
    assert(f != nullptr);
    assert(f == b);  // reuses the same chunk

    // Pool is exhausted again
    assert(pool.allocate() == nullptr);

    // Return all and re-allocate
    pool.deallocate(a);
    pool.deallocate(c);
    pool.deallocate(d);
    pool.deallocate(f);

    // Should be able to allocate 4 again
    for (int i = 0; i < 4; ++i) {
        assert(pool.allocate() != nullptr);
    }
    assert(pool.allocate() == nullptr);

    std::cout << "All raw MemoryPool tests passed.\n\n";
}
```