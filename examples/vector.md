# Vector

## Allocation and Alignment

`Vector` must handle objects that are not default-constructible. Having a `T*` array inside would prevent preallocation, because object would have to be constructed using their default constructors. This is why a `char*` or `std::byte*` is used instead, but this introduces some complexity. The array must be aligned when allocated so that it corresponds to the alignment requirements of the `T`. As for dynamically allocated arrays, `::operator new(size, std::align_val_t{alignof(T)})` or `std::aligned_alloc(alignof(T), size)` can be used. When `::operator new(size, align)` is used to allocate the memory, the corresponding `::operator delete(ptr, align)` must be used to free the memory.

`new char[n]` / `delete[]` is a two-step process under the hood: allocate raw memory, then construct `n` `char` objects (trivial in this case), and `delete[]` destructs them all then frees. The runtime needs to track the array length somewhere (often stored just before the returned pointer) to know how many destructors to call.

`::operator new(size)` / `::operator delete(ptr)` (and the aligned variants) is **just malloc/free with extra steps**. It gives you `size` bytes of raw memory back. No constructors, no destructors, no array bookkeeping. You're telling the runtime "here's a block of memory, free it," and that's all it needs- the allocator already knows the block size from its own internal bookkeeping (same way `free` works in C without knowing the size).

So the pairing is:

```
new char[n]                    <->  delete[] ptr
::operator new(size)           <->  ::operator delete(ptr)
::operator new(size, align)    <->  ::operator delete(ptr, align)
```

Mixing them is undefined behavior.

## Strong Exception Guarantee

*If the function throws an exception, the state of the program is rolled back to the state just before the function call*

This means that if `push_back` were to fail, the state of the program must be rolled back to the state before the `push_back` was called. This is why `push_back`'s reallocation operates as follows:

- a new, properly aligned memory is allocated with increased capacity
- since this is a byte array, there are no objects *alive* in the array
    - the memory region is **reinterpreted** as if it was an object of type `T`
    - since the original pointer did not point to a `T` object, it must be **laundered**
    - since the object's lifetime did not yet start, **placement new** must be used to create it (calling copy or move assignment is UB)
- strong exception guarantees must be respected, therefore
    - **move** can only be called if the move constructor is **noexcept**, otherwise it could leave half-moved-from objects behind in the original buffer
    - if a copy constructor **throws**
        - all of the newly constructed objects must be destroyed by calling their **destructors**
        - the allocated new buffer myst be **deleted**
        - the exception must be **thrown** to be handled by the user
- when all of the new objects have been **successfully constructed**
    - the **destructors** of the original objects must be called
    - the original buffer must be **deleted**
    - the **pointers** must be swapped
    - stored **capacity** should be increased

## Implementation

```cpp
#include <cstddef>
#include <cstring>
#include <new>
#include <utility>

template <typename T>
class Vector {
private:
    char*       m_data     = nullptr; // maybe std::byte is better in such situations
    std::size_t m_size     = 0;
    std::size_t m_capacity = 0;

    T* slot(std::size_t i) {
        return std::launder(reinterpret_cast<T*>(m_data + i * sizeof(T)));
    }

    const T* slot(std::size_t i) const {
        return std::launder(reinterpret_cast<const T*>(m_data + i * sizeof(T)));
    }

    void reallocate(std::size_t new_cap) {
        void* raw = ::operator new(new_cap * sizeof(T), std::align_val_t{alignof(T)});
        char* new_data = static_cast<char*>(raw);

        std::size_t constructed = 0;
        try {
            for (; constructed < m_size; ++constructed) {
                T* src = slot(constructed);
                T* dst = reinterpret_cast<T*>(new_data + constructed * sizeof(T));

                // Copy if move might throw - originals stay intact on failure.
                // Move if noexcept - safe and faster.
                new (dst) T(std::move_if_noexcept(*src));
            }
        }
        catch (...) {
            // Undo what we built in the new buffer
            for (std::size_t j = 0; j < constructed; ++j) {
                reinterpret_cast<T*>(new_data + j * sizeof(T))->~T();
            }
            ::operator delete(new_data, std::align_val_t{alignof(T)});
            throw;  // originals in m_data are untouched (we copied, not moved)
        }

        // Success - destroy originals, swap buffers
        for (std::size_t i = 0; i < m_size; ++i) {
            slot(i)->~T();
        }
        ::operator delete(m_data, std::align_val_t{alignof(T)});

        m_data     = new_data;
        m_capacity = new_cap;
    }

    void destroy_all() {
        for (std::size_t i = 0; i < m_size; ++i) {
            slot(i)->~T();
        }
        ::operator delete(m_data, std::align_val_t{alignof(T)});
        m_data     = nullptr;
        m_size     = 0;
        m_capacity = 0;
    }

public:
    Vector() = default;
    ~Vector() { destroy_all(); }

    std::size_t size()     const { return m_size; }
    std::size_t capacity() const { return m_capacity; }

    T&       operator[](std::size_t i)       { return *slot(i); }
    const T& operator[](std::size_t i) const { return *slot(i); }

    void push_back(const T& value) {
        if (m_size == m_capacity) {
            reallocate(m_capacity == 0 ? 1 : m_capacity * 2);
        }

        // Attempt the copy-construct into the slot.
        // If it throws, m_size hasn't been bumped yet, so the
        // vector's state is exactly as it was before the call -
        // strong exception guarantee.
        new (m_data + m_size * sizeof(T)) T(value);

        // Only commit after construction succeeds.
        ++m_size;
    }

    void push_back(T&& value) {
        if (m_size == m_capacity) {
            reallocate(m_capacity == 0 ? 1 : m_capacity * 2);
        }

        new (m_data + m_size * sizeof(T)) T(std::move(value));
        ++m_size;
    }

    void pop_back() {
        --m_size;
        slot(m_size)->~T();
    }
};
```