# new

---

## The Two Things `new` Does

This is the most important thing to understand. A `new` expression always does **two separate steps**:

1. **Allocate memory** -- calls an `operator new` function to get raw bytes.
2. **Construct the object** -- calls the constructor at that memory address.

`delete` does the reverse: destruct, then deallocate. Understanding that these are *two separate operations bundled together* is the key to understanding everything below.

---

## 01 - Regular `new` (The Everyday Form)

```cpp
Widget* ptr = new Widget(42);
```

What happens:
1. Calls `operator new(sizeof(Widget))` to allocate memory.
2. Calls `Widget(42)` constructor at that address.
3. If the constructor throws, the matching `operator delete` is called automatically.

Deallocation:
```cpp
delete ptr;   // calls ~Widget(), then operator delete(ptr)
```

For arrays:
```cpp
Widget* arr = new Widget[10];   // calls operator new[], then 10x default ctor
delete[] arr;                   // calls 10x ~Widget(), then operator delete[]
```

**Always match `new` with `delete` and `new[]` with `delete[]`.** Mixing them is undefined behavior.

---

## 02 - Global `operator new` (The Allocation Function)

`operator new` is just a function that allocates raw memory. The global one lives in the `std` namespace and behaves like a smarter `std::malloc`:

```cpp
// These are the global allocation functions the compiler provides.
// You can call them directly, but you almost never should.
void* operator new(std::size_t size);                  // throwing
void* operator new(std::size_t size, std::nothrow_t);  // non-throwing
void  operator delete(void* ptr) noexcept;
void  operator delete(void* ptr, std::size_t size) noexcept;  // C++14 sized
```

The throwing version throws `std::bad_alloc` on failure. The `nothrow` version returns `nullptr`:

```cpp
// Throwing (default) -- throws std::bad_alloc on failure
// new expression: allocates + constructs
Widget* a = new Widget();

// Non-throwing -- returns nullptr on failure
// new expression: allocates + constructs
Widget* b = new (std::nothrow) Widget();
if (b == nullptr) {
    // allocation failed
}

// operator new function call: allocates only
void* raw = operator new(sizeof(Widget));  // you get raw bytes, no Widget
```

### Replacing the global operator new

You can replace the global `operator new` to hook into all heap allocations in your program. This is useful for custom allocators, leak tracking, or profiling:

```cpp
// This replaces the global allocator for the entire program.
// No header needed -- the compiler recognizes these signatures.
void* operator new(std::size_t size) {
    std::cout << "allocating " << size << " bytes" << std::endl;
    void* ptr = std::malloc(size);
    if (ptr == nullptr) {
        throw std::bad_alloc();
    }
    return ptr;
}

void operator delete(void* ptr) noexcept {
    std::cout << "freeing memory" << std::endl;
    std::free(ptr);
}
```

After this, every `new` expression in the entire program (including inside the standard library) will go through your function.

---

## 03 - Class-Specific `operator new`

A class can provide its own allocation function. The compiler prefers the class-specific version over the global one:

```cpp
class Widget {
public:
    static void* operator new(std::size_t size) {
        return std::malloc(size);
    }

    static void operator delete(void* ptr) noexcept {
        std::free(ptr);
    }
};

Widget* w = new Widget();    // calls Widget::operator new
delete w;                    // calls Widget::operator delete

Widget* g = ::new Widget();  // calls global ::operator new, bypasses the class one
::delete g;                  // calls global ::operator delete
```

### Why do this?

The main use case is **memory pools**. If you know you'll create millions of small `Widget` objects, you can allocate a large block up front and hand out fixed-size chunks, avoiding the overhead of calling `std::malloc` millions of times:

```cpp
class Widget {
public:
    static void* operator new(std::size_t size) {
        return pool_allocate(size);  // fast, no fragmentation
    }

    static void operator delete(void* ptr) noexcept {
        pool_deallocate(ptr);
    }
private:
    // ... pool implementation ...
};
```

### Lookup order

When the compiler sees `new Widget()`:
1. Look for `Widget::operator new`.
2. If not found, look for `BaseClass::operator new` (walks up the hierarchy).
3. If not found, use `::operator new` (global).

This is why `::new` exists -- it skips steps 1 and 2 entirely.

---

## 04 - Placement `new` (Construct at a Specific Address)

Placement `new` separates the two steps: you provide the memory, and `new` only does the construction part.

```cpp
#include <new>  // required for placement new

alignas(Widget) std::uint8_t buf[sizeof(Widget)];

// Step 1: memory is already provided (buf on the stack)
// Step 2: construct a Widget at that address
Widget* w = ::new (buf) Widget(42);

// You must manually destroy -- there is no "placement delete"
w->~Widget();

// Do NOT call delete w -- you didn't allocate with regular new
```

### How it works internally

The global placement `operator new` is trivial -- it just returns the pointer you gave it:

```cpp
// This is what the standard library provides.
// It is guaranteed to be a no-op.
void* operator new(std::size_t /*size*/, void* ptr) noexcept {
    return ptr;
}
```

So `::new (buf) Widget(42)` effectively does:
1. Call `operator new(sizeof(Widget), buf)` -> returns `buf` unchanged.
2. Call `Widget(42)` constructor at `buf`.

That is why it is purely a construction mechanism.

### The `::new` habit

A class could theoretically overload placement `operator new`:

```cpp
class Widget {
public:
    // Unusual, but legal
    static void* operator new(std::size_t size, void* ptr) {
        std::cout << "Widget's custom placement new?!" << std::endl;
        return ptr;
    }
};

new (buf) Widget();    // calls Widget::operator new(size, buf) -- the custom one
::new (buf) Widget();  // calls ::operator new(size, buf) -- the standard no-op
```

In systems code, you always write `::new` for placement new to guarantee the standard behavior.

---

## 05 - Alignment-Aware `new` (C++17)

Before C++17, `operator new` only guaranteed alignment up to `__STDCPP_DEFAULT_NEW_ALIGNMENT__` (typically 16 bytes). Over-aligned types were a problem.

C++17 added alignment-aware overloads that the compiler calls automatically:

```cpp
struct alignas(64) CacheLine {
    std::uint8_t m_data[64];
};

// Pre-C++17: this might return a 16-byte-aligned address -- UB!
// C++17: compiler calls operator new(size, align_val_t{64}) -- correct.
CacheLine* ptr = new CacheLine();
delete ptr;
```

The new overloads:

```cpp
void* operator new(std::size_t size, std::align_val_t align);
void  operator delete(void* ptr, std::size_t size, std::align_val_t align);
```

You can also override these per-class for aligned pool allocators:

```cpp
class alignas(64) CacheLine {
public:
    static void* operator new(std::size_t size, std::align_val_t align) {
        return std::aligned_alloc(static_cast<std::size_t>(align), size);
    }

    static void operator delete(void* ptr, std::size_t /*size*/, std::align_val_t /*align*/) noexcept {
        std::free(ptr);
    }
};
```

**Note**: What are over-aligned types? Most types have an alignment requirement that's equal to their size or some natural boundary: `int` needs 4 bytes, `double` needs 8, and so on. The platform defines a "default" maximum alignment that the standard allocator guarantees, which is `__STDCPP_DEFAULT_NEW_ALIGNMENT__`, typically 16 bytes on most 64-bit systems. An over-aligned type is any type whose alignment requirement exceeds that default. You create them explicitly with `alignas`. Why would you want this? The main reasons are cache line alignment and SIMD/hardware requirements. A CPU cache line is typically 64 bytes. If you want to guarantee that a struct sits on its own cache line (to avoid false sharing between threads, for example), you align it to 64. SIMD intrinsics (SSE, AVX) require 16-byte or 32-byte alignment for their vector types.

---

## 06 - `new` and Exceptions

This is an important detail that `new` handles for you. If the constructor throws after memory is allocated, the matching `operator delete` is called automatically:

```cpp
// If Widget(42) throws:
// 1. operator new(sizeof(Widget)) was already called -- memory is allocated.
// 2. The compiler automatically calls operator delete(ptr) to free it.
// No memory leak.
Widget* w = new Widget(42);
```

With placement new, this means the placement `operator delete` is called (which is also a no-op):

```cpp
// If Widget(42) throws here:
// 1. Placement operator new returned buf -- no real allocation happened.
// 2. Compiler calls placement operator delete -- also a no-op.
// 3. buf is still just a byte array on the stack.
// No leak, no problem.
Widget* w = ::new (buf) Widget(42);
```

This is one reason you cannot just use `operator new` and constructor calls separately -- the `new` expression handles the exception safety dance for you.

---

## 07 - Summary: Which `new` Does What

| Syntax | Allocates? | Constructs? | Use case |
|---|---|---|---|
| `new Widget()` | Yes (heap) | Yes | Everyday heap allocation |
| `new (std::nothrow) Widget()` | Yes (heap, nullptr on fail) | Yes | When you want to check for null |
| `::new Widget()` | Yes (heap, global) | Yes | Bypass class-specific allocator |
| `::new (buf) Widget()` | No (you provide memory) | Yes | Placement new -- pools, arenas |
| `operator new(size)` | Yes (heap) | No | Raw allocation only (rare) |

And the matching cleanup:

| Allocation | Cleanup |
|---|---|
| `new Widget()` | `delete ptr;` |
| `new Widget[n]` | `delete[] ptr;` |
| `::new (buf) Widget()` | `ptr->~Widget();` (manual dtor, no delete) |
| `operator new(size)` | `operator delete(ptr);` |

---

## 08 - Common Pitfalls

### Mixing `new`/`delete` with `new[]`/`delete[]`

```cpp
Widget* arr = new Widget[10];
delete arr;     // UB -- must use delete[]
```

### Calling `delete` on a placement-new object

```cpp
alignas(Widget) std::uint8_t buf[sizeof(Widget)];
Widget* w = ::new (buf) Widget();
delete w;       // UB -- buf is on the stack, not from the heap allocator
```

### Forgetting the destructor with placement new

```cpp
void* raw = std::malloc(sizeof(Widget));
Widget* w = ::new (raw) Widget();
std::free(raw); // Skips ~Widget() -- resources may leak
// Correct:
// w->~Widget();
// std::free(raw);
```

### Class `operator new` with inheritance

```cpp
class Base {
public:
    static void* operator new(std::size_t size) {
        // Might assume size == sizeof(Base)
        return pool_allocate(size);  // bug if Derived is larger!
    }
};

class Derived : public Base {
    int m_extra[100];
};

// Calls Base::operator new with sizeof(Derived) -- your pool
// might not handle this correctly if it assumes fixed-size blocks.
Derived* d = new Derived();
```

Always check the `size` parameter in class-specific `operator new` if the class might be inherited from.