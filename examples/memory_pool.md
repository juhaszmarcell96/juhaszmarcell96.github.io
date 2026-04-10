# Bump/Arena Allocator

It allocates one big byte buffer up front, then hands out chunks sequentially by bumping an offset forward. Individual objects can't be freed - you free everything at once with `clear()`. This pattern is common in game engines, frame-based systems, and high-frequency trading where allocation speed matters and per-object `delete` is unnecessary.

## The Memory Layout

Each `create()` call produces this in the buffer:

```
[padding] [Header] [padding] [Object] [padding] [Header] [padding] [Object] ...
          ^                  ^                   ^                  ^
          hdr_ptr            obj_ptr             next hdr_ptr       next obj_ptr
```

The padding comes from alignment. `std::align` adjusts the pointer forward to meet the alignment requirement of whatever comes next.

---

```cpp
#include <cstddef>
#include <cstdint>
#include <limits>
#include <new>
#include <memory>
#include <type_traits>
#include <utility>

class MemoryPool {
private:
    static constexpr std::size_t k_no_header = std::numeric_limits<std::size_t>::max();
    
    struct Header {
        void (*destroy)(void*);        // type-erased destructor (nullptr if trivial)
        std::size_t object_offset;     // where the object lives in buffer
        std::size_t next_offset;       // where the next header region starts
        std::size_t prev_offset;       // where the previous header region starts
    };

    std::unique_ptr<std::byte[]> m_buffer;
    std::size_t m_capacity;
    std::size_t m_offset;
    std::size_t m_last_header_offset;
public:
    explicit MemoryPool(std::size_t capacity)
        : m_buffer(new std::byte[capacity])
        , m_capacity(capacity)
        , m_offset(0)
        , m_last_header_offset(k_no_header)
    {}

    ~MemoryPool() { clear(); }

    MemoryPool(const MemoryPool&) = delete;
    MemoryPool& operator=(const MemoryPool&) = delete;

    template <typename T, typename... Args>
    [[nodiscard]] T* create(Args&&... args) {
        // 1. Align and place the header
        void* hdr_ptr = m_buffer.get() + m_offset;
        std::size_t remaining = m_capacity - m_offset;

        if (!std::align(alignof(Header), sizeof(Header), hdr_ptr, remaining)) {
            throw std::bad_alloc{};
        }

        std::size_t header_pos = static_cast<std::byte*>(hdr_ptr) - m_buffer.get();
        std::size_t after_header = header_pos + sizeof(Header);

        // 2. Align and place the object after the header
        void* obj_ptr = m_buffer.get() + after_header;
        remaining = m_capacity - after_header;

        if (!std::align(alignof(T), sizeof(T), obj_ptr, remaining)) {
            throw std::bad_alloc{};
        }

        std::size_t object_pos = static_cast<std::byte*>(obj_ptr) - m_buffer.get();
        std::size_t after_object = object_pos + sizeof(T);

        // 3. Write the header
        Header* header = ::new (hdr_ptr) Header{};
        header->object_offset = object_pos;
        header->next_offset = after_object;
        header->prev_offset = m_last_header_offset;

        if constexpr (!std::is_trivially_destructible_v<T>) {
            header->destroy = [](void* p) {
                // the pointer we get here is probably a calculated one -> std::launder
                std::launder(static_cast<T*>(p))->~T();
            };
        }
        else {
            header->destroy = nullptr;
        }

        // 4. Construct the object
        T* obj = ::new (obj_ptr) T(std::forward<Args>(args)...);

        // 5. Update pool state (only after everything succeeded)
        // It is important to only update the offset afterwards, because it can happen that
        // the constructor of T throws. In that case, although the header is already written
        // the offset is not adjusted, so it will be overwritten with the next create call
        // and clear will not reach it.
        m_last_header_offset = header_pos;
        m_offset = after_object;

        return obj;
    }

    void clear() {
        // Walk backward through headers, destroying in reverse creation order
        std::size_t current = m_last_header_offset;

        while (current != k_no_header) {
            void* hdr_ptr = m_buffer.get() + current;
            // The header was placed by placement new, but this pointer is not the one
            // that was returned by the placement new call, it is a freshly computed one.
            // To make sure that the compiler knows that a header object lives there,
            // it is important to use std::launder.
            Header* header = std::launder(static_cast<Header*>(hdr_ptr));

            if (header->destroy != nullptr) {
                void* obj_ptr = m_buffer.get() + header->object_offset;
                header->destroy(obj_ptr);
            }

            current = header->prev_offset;
        }

        m_offset = 0;
        m_last_header_offset = k_no_header;
    }
};
```

---

## Key Design Decisions

### The Header stores a type-erased destructor

```cpp
void (*destroy)(void*);   // function pointer: takes void*, calls the real dtor
```

This is the critical trick. The pool stores heterogeneous types - you can do `create<Widget>()` followed by `create<Config>()` - but `clear()` doesn't know about those types. The lambda captures the type at `create` time:

```cpp
header->destroy = [](void* p) { static_cast<T*>(p)->~T(); };
```

This is a captureless lambda, so it decays to a plain function pointer. No heap allocation for the lambda itself.

### The `if constexpr` optimization

```cpp
if constexpr (!std::is_trivially_destructible_v<T>)
```

If `T` has a trivial destructor (like `int`, `double`, POD structs), there's nothing to call, so the function pointer is set to `nullptr`. `clear()` checks for `nullptr` before calling. This avoids paying for a function pointer call on types that don't need it.

### Copy/move are deleted

```cpp
MemoryPool(const MemoryPool&) = delete;
MemoryPool& operator=(const MemoryPool&) = delete;
```

Correct - copying a pool makes no sense (the objects inside hold raw pointers into the buffer), and move isn't implemented. You could add move support, but it's tricky because any pointers returned by `create()` would dangle if the buffer moves.

## How `std::align` Works

This is worth understanding on its own since it shows up in the code twice:

```cpp
void* ptr = m_buffer.get() + m_offset;
std::size_t remaining = m_capacity - m_offset;

std::align(alignof(T), sizeof(T), ptr, remaining);
// ptr is adjusted FORWARD to the next aligned address
// remaining is decreased by the padding consumed
// returns nullptr if there's not enough space
```

It doesn't allocate anything. It just bumps `ptr` forward to satisfy the alignment and tells you how much space is left.

## Things to Think About

### Destruction order

The comment in `clear()` acknowledges this: objects are destroyed **front-to-back** (in creation order). Normal C++ semantics destroy in **reverse** order (last created, first destroyed). If objects reference each other, this could matter. A production-quality version would walk the headers to collect records, then destroy in reverse.

### No individual deallocation

This is by design. If you need per-object `delete`, you want a different allocator (free-list, slab allocator). The tradeoff is speed - bump allocation is essentially just incrementing an integer.

### Thread safety

There's none. If two threads call `create()` concurrently, they'll race on `m_offset`. You'd need a `std::atomic<std::size_t>` with `fetch_add` for a lock-free version, or a mutex for simplicity.

### The `next_offset` tracks pre-alignment position

```cpp
header->next_offset = after_object;  // next header starts here (before alignment)
```

This is important - `next_offset` points to right after the object, not to the next aligned header position. The alignment adjustment happens when `clear()` calls `std::align` during its walk. This avoids wasting the padding twice (once in the header, once during traversal).