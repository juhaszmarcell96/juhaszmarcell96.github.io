# PMR (Polymorphic Memory Resource) Allocators

`std::pmr` (C++17) lets you swap allocation strategies at runtime without changing
container types. The key class is `std::pmr::memory_resource` - you subclass it or
use the built-in ones.

## Using the built-in PMR allocators

```cpp
#include <array>
#include <cstddef>
#include <iostream>
#include <memory_resource>
#include <string>
#include <vector>

int main() {
    // -- monotonic_buffer_resource (arena / bump allocator) --
    // Pre-allocate a stack buffer. The allocator bumps a pointer forward.
    // Never frees individual allocations. Frees everything on destruction.
    // Extremely fast: O(1) allocation, zero fragmentation.

    std::array<std::byte, 4096> buffer{};
    std::pmr::monotonic_buffer_resource arena(
        buffer.data(), buffer.size(),
        std::pmr::null_memory_resource()  // upstream: fail if buffer exhausted
    );

    // A vector that allocates from our arena
    std::pmr::vector<int> numbers(&arena);
    numbers.reserve(100);
    for (int i = 0; i < 100; ++i) {
        numbers.push_back(i);
    }
    // All 100 ints live inside 'buffer' on the stack. Zero heap allocations.

    // -- unsynchronized_pool_resource (pool allocator) --
    // Maintains pools of different block sizes. Good for heterogeneous
    // allocations. NOT thread-safe (use synchronized_pool_resource for that).

    std::pmr::unsynchronized_pool_resource pool;
    std::pmr::vector<std::pmr::string> names(&pool);
    names.emplace_back("Alice");
    names.emplace_back("Bob");
    // Both the vector's internal array and the strings' character data
    // are allocated from the pool.

    // -- Chaining: pool backed by arena --
    // If pool runs out, it asks the arena. If the arena runs out, it fails.
    std::array<std::byte, 65536> big_buffer{};
    std::pmr::monotonic_buffer_resource backing_arena(
        big_buffer.data(), big_buffer.size(),
        std::pmr::null_memory_resource()
    );
    std::pmr::unsynchronized_pool_resource chained_pool(&backing_arena);

    std::pmr::vector<std::pmr::string> data(&chained_pool);
    for (int i = 0; i < 50; ++i) {
        data.emplace_back("item_" + std::to_string(i));
    }
    // Everything lives in big_buffer. No malloc/new calls at all.

    std::cout << "PMR examples completed. numbers.size() = "
              << numbers.size() << "\n";
    return 0;
}
```

## Writing a custom memory resource

```cpp
#include <cstddef>
#include <cstdlib>
#include <iostream>
#include <memory_resource>
#include <vector>

// A logging allocator that wraps another resource and prints every
// allocation/deallocation. Useful for debugging.
class LoggingResource : public std::pmr::memory_resource {
private:
    std::pmr::memory_resource* m_upstream;
    std::size_t m_total_allocated = 0;

    [[nodiscard]]
    void* do_allocate(std::size_t bytes, std::size_t alignment) override {
        void* ptr = m_upstream->allocate(bytes, alignment);
        m_total_allocated += bytes;
        std::cout << "  [alloc]   " << bytes << " bytes, align "
                  << alignment << " -> " << ptr << "\n";
        return ptr;
    }

    void do_deallocate(void* ptr, std::size_t bytes, std::size_t alignment) override {
        std::cout << "  [dealloc] " << bytes << " bytes <- " << ptr << "\n";
        m_upstream->deallocate(ptr, bytes, alignment);
    }

    [[nodiscard]]
    bool do_is_equal(const std::pmr::memory_resource& other) const noexcept override {
        return this == &other;
    }

public:
    explicit LoggingResource(
        std::pmr::memory_resource* upstream = std::pmr::get_default_resource())
        : m_upstream(upstream)
    {}

    [[nodiscard]]
    std::size_t total_allocated() const noexcept { return m_total_allocated; }
};

int main() {
    LoggingResource logger;
    std::pmr::vector<int> v(&logger);

    std::cout << "Pushing 5 ints:\n";
    for (int i = 0; i < 5; ++i) {
        v.push_back(i);
    }

    std::cout << "\nTotal allocated: " << logger.total_allocated() << " bytes\n";
    std::cout << "\nDestroying vector:\n";
    // Destructor runs here -- you'll see the deallocations
    return 0;
}
```

### PMR Allocators
- `monotonic_buffer_resource` -- bump allocator, fastest possible, bulk-free only.
- `unsynchronized_pool_resource` -- size-segregated pools, individual free.
- `synchronized_pool_resource` -- thread-safe version.
- Chain them: pool backed by arena backed by `null_memory_resource`.
- Subclass `memory_resource` and override `do_allocate`, `do_deallocate`, `do_is_equal`.