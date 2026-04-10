# UniquePtr

Here's a modern C++ `unique_ptr` implementation covering the core functionality:

```cpp
#include <cstddef>
#include <utility>

template <typename T>
class UniquePtr {
private:
    T* m_ptr;
public:
    // -- constructors --
    constexpr UniquePtr() noexcept : m_ptr(nullptr) {}
    constexpr explicit UniquePtr(T* ptr) noexcept : m_ptr(ptr) {}
    constexpr UniquePtr(std::nullptr_t) noexcept : m_ptr(nullptr) {}

    // -- no copy --
    UniquePtr(const UniquePtr&) = delete;
    UniquePtr& operator=(const UniquePtr&) = delete;

    // -- move --
    constexpr UniquePtr(UniquePtr&& other) noexcept : m_ptr(other.release()) {}
    constexpr UniquePtr& operator=(UniquePtr&& other) noexcept {
        // this self-assignment check would not be absolutely necessary
        // it would be correct without it as well
        if (this != &other) {
            reset(other.release());
        }
        return *this;
    }

    // Converting move constructor: UniquePtr<Derived> -> UniquePtr<Base>
    // Only enabled when U* is implicitly convertible to T*
    // (e.g. Derived* -> Base*, but not Base* -> Derived*)
    template <typename U, typename = std::enable_if_t<std::is_convertible_v<U*, T*>>>
    constexpr UniquePtr(UniquePtr<U>&& other) noexcept : m_ptr(other.release()) {}

    // Converting move assignment
    template <typename U, typename = std::enable_if_t<std::is_convertible_v<U*, T*>>>
    constexpr UniquePtr& operator=(UniquePtr<U>&& other) noexcept {
        reset(other.release());
        return *this;
    }

    // -- destructor --
    ~UniquePtr() noexcept { delete m_ptr; }

    // -- modifiers --
    [[nodiscard]] constexpr T* release() noexcept {
        T* old = m_ptr;
        m_ptr = nullptr;
        return old;
    }

    constexpr void reset(T* ptr = nullptr) noexcept {
        // Assign before delete: if the old object's destructor somehow
        // accesses this UniquePtr (e.g. via a back-reference), it will
        // see the new pointer, not a dangling one.
        T* old = m_ptr;
        m_ptr = ptr;
        delete old;
    }

    constexpr void swap(UniquePtr& other) noexcept {
        T* tmp = m_ptr;
        m_ptr = other.m_ptr;
        other.m_ptr = tmp;
    }

    // -- observers --
    [[nodiscard]] constexpr T* get() const noexcept { return m_ptr; } 
    [[nodiscard]] constexpr explicit operator bool() const noexcept { return m_ptr != nullptr; }
    [[nodiscard]] constexpr T& operator*() const noexcept { return *m_ptr; }
    [[nodiscard]] constexpr T* operator->() const noexcept { return m_ptr; }
};

// -- make_unique --

template <typename T, typename... Args>
[[nodiscard]] constexpr UniquePtr<T> make_unique(Args&&... args) {
    return UniquePtr<T>(new T(std::forward<Args>(args)...));
}

// -- comparison operators --
template <typename T>
[[nodiscard]] constexpr bool operator==(const UniquePtr<T>& lhs, std::nullptr_t) noexcept {
    return !lhs;
}

template <typename T>
[[nodiscard]] constexpr bool operator==(std::nullptr_t, const UniquePtr<T>& rhs) noexcept {
    return !rhs;
}
```

**RAII** is the central idea - the constructor acquires, the destructor releases, and the class is non-copyable to enforce unique ownership.

**Why `release()` in the move constructor** rather than just swapping: it leaves the source in a clean null state without needing to delete anything, which is simpler and avoids a redundant branch.

**`reset()` deletes the old pointer *after* reassigning** - this ordering matters if `T`'s destructor somehow interacts with this `UniquePtr` (a paranoid but real concern in production code).

**What's omitted** for brevity: custom deleter support (the second template parameter) and array specialization (`UniquePtr<T[]>`). The array support can be added by adding a partial specialization of the entire class for `T[]`. It changes three things: uses `delete[]` instead of `delete`, provides `operator[]` instead of `operator*` / `operator->`, and `make_unique` allocates an array.

```cpp
template <typename T>
class UniquePtr<T[]> {
public:
    // ...

    // -- destructor --
    // The only fundamental difference: delete[] instead of delete

    ~UniquePtr() noexcept {
        delete[] m_ptr;
    }

    // ...

    constexpr void reset(T* ptr = nullptr) noexcept
    {
        T* old = m_ptr;
        m_ptr = ptr;
        delete[] old;
    }

    // ...

    // No operator* or operator-> for arrays.
    // Instead, provide operator[] for element access.
    [[nodiscard]] constexpr T& operator[](std::size_t index) const noexcept {
        return m_ptr[index];
    }

    // ...
};

// Single object (existing)
template <typename T, typename... Args>
[[nodiscard]] constexpr UniquePtr<T> make_unique(Args&&... args) {
    return UniquePtr<T>(new T(std::forward<Args>(args)...));
}

// Array version: make_unique<int[]>(10) allocates 10 value-initialized ints.
// The enable_if prevents this from matching non-array types.
template <typename T>
[[nodiscard]] constexpr std::enable_if_t<std::is_array_v<T>, UniquePtr<T>>
make_unique(std::size_t size) {
    // std::remove_extent_t<int[]> is int
    using Elem = std::remove_extent_t<T>;
    return UniquePtr<T>(new Elem[size]());  // () for value initialization (zero-init)
}
```

---

## What the Array Specialization Does NOT Support

**No converting constructor.** Unlike the single-object version, `UniquePtr<Base[]>` cannot accept `UniquePtr<Derived[]>`. Polymorphic arrays are broken in C++ because `delete[]` needs to know the element size, and `Base[]` and `Derived[]` have different element sizes. The standard `std::unique_ptr<T[]>` also blocks this.

**No size tracking.** The pointer itself doesn't know how many elements were allocated, just like raw `new[]`. If you need bounds checking, you want `std::vector` or `std::span` on top.

---

## Disambiguating the Two `make_unique` Overloads

The two overloads need to not interfere with each other. The array version uses `std::enable_if_t<std::is_array_v<T>>` as the return type, so it only participates when `T` is an array type. The single-object version only matches non-array types because `UniquePtr<T>` for a non-array `T` is the primary template, not the specialization.

If you wanted to be extra safe, you could guard the single-object version too:

```cpp
template <typename T, typename... Args>
[[nodiscard]] constexpr std::enable_if_t<!std::is_array_v<T>, UniquePtr<T>>
make_unique(Args&&... args)
{
    return UniquePtr<T>(new T(std::forward<Args>(args)...));
}
```

This prevents someone from accidentally writing `make_unique<int[]>(42)` and hitting the wrong overload.