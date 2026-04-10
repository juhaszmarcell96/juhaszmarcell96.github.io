# C++ Constructors vs Assignment

When each is called, how to force construction over assignment, and safe patterns for preallocated memory.

---

## 01 - Quick Reference

| Expression | What is called | Type |
|---|---|---|
| `Widget w;` | Default constructor | CTOR |
| `Widget w(other);` | Copy constructor | CTOR |
| `Widget w = other;` | Copy constructor (NOT assignment!) | CTOR |
| `Widget w(std::move(other));` | Move constructor | MOVE CTOR |
| `Widget w = std::move(other);` | Move constructor (NOT assignment!) | MOVE CTOR |
| `w = other;` (w already exists) | Copy assignment operator | ASSIGN |
| `w = std::move(other);` (w already exists) | Move assignment operator | ASSIGN |

**The rule:** If the object being written to is being *created in this statement*, a constructor is called. If it *already exists*, an assignment operator is called.

---

## 02 - A Minimal Class to Trace Calls

```cpp
class Widget {
public:
    Widget() { }                                  // default constructor
    Widget(const Widget& other) { }               // copy constructor
    Widget(Widget&& other) noexcept { }           // move constructor
    Widget& operator=(const Widget& other) {      // copy assignment (const & input)
        if (this != &other) { /* do stuff */ }    //     - check whether it is self-assignment
        return *this;
    }
    Widget& operator=(Widget&& other) noexcept {  // move assignment (non-const && input)
        if (this != &other) { /* do stuff */ }    //     - check whether it is self-assignment
        return *this;
    }
    ~Widget() { }                                 // destructor
};
```

---

## 03 - Constructor vs Assignment in Action

```cpp
int main() {
    Widget a;                       // default ctor
    Widget b = a;                   // copy ctor (initialization, NOT assignment!)
    Widget c(a);                    // copy ctor (direct initialization)
    Widget d = std::move(a);        // move ctor (initialization with rvalue)
    Widget e(std::move(b));         // move ctor (direct initialization)

    c = a;                          // copy assign (c already exists)
    d = std::move(a);              // move assign (d already exists)

    return 0;
}
```

**Key insight:** `Widget b = a;` looks like assignment because of the `=` sign, but it is copy *construction*. The `=` in a declaration is initialization syntax, not the assignment operator.

---

## 04 - The Preallocated Memory Problem

When you allocate raw memory (e.g. via `std::malloc`, `operator new`, or a memory pool), no object exists there yet. The bytes are uninitialized. Casting that pointer and calling assignment is undefined behavior:

```cpp
// --- DANGEROUS: assignment on unconstructed memory ---
void* raw = std::malloc(sizeof(Widget));

// This is undefined behavior!
// No Widget object exists at 'raw' yet.
// The assignment operator reads from *this (garbage) and
// may free resources that were never allocated.
auto* ptr = static_cast<Widget*>(raw);
*ptr = some_widget;                  // <-- UB: calls operator= on non-object

std::free(raw);                      // <-- also UB: no dtor called
```

**Why this is dangerous:** The assignment operator assumes `*this` is a valid, fully constructed object. It might try to release old resources (e.g. `delete m_data`) that point to garbage. It might read a vtable pointer that is uninitialized. This is UB even if it "seems to work."

---

## 05 - Placement New -- The Safe Way

Placement `new` constructs an object at a specific memory address. It calls the constructor, not the assignment operator, which is exactly what you need for raw memory.

```cpp
#include <new>       // required for placement new
#include <cstdlib>
#include <memory>

// --- SAFE: placement new for copy construction ---
void* raw = std::malloc(sizeof(Widget));

Widget source;

// Copy-construct a Widget at 'raw'
Widget* ptr = ::new (raw) Widget(source);                // COPY CTOR

// Or move-construct a Widget at 'raw'
// Widget* ptr = ::new (raw) Widget(std::move(source));  // MOVE CTOR

// Best of both wolds: MOVE CTOR if it is noexcept, COPY CTOR otherwise
// Widget* ptr = ::new (raw) Widget(std::move_if_noexcept(source));

// Must manually call destructor (placement new = manual lifetime)
ptr->~Widget();
std::free(raw);
```

**This is correct:** Placement `new` starts the object's lifetime at that address. The constructor initializes all members properly, and the vtable pointer (if any) is set up.

---

## 06 - Placement New for Default + Move-Assign Pattern

Sometimes you need to default-construct first and assign later (e.g. a pool of reusable slots). That is fine, as long as the object is *constructed first*:

```cpp
// Allocate a pool of N slots
constexpr std::size_t pool_size = 8;
alignas(Widget) std::uint8_t pool[pool_size * sizeof(Widget)];

// Default-construct all slots up front
for (std::size_t i = 0; i < pool_size; ++i) {
    ::new (&pool[i * sizeof(Widget)]) Widget();     // DEFAULT CTOR
}

// Now assignment is safe -- the objects exist
auto* slot_3 = std::launder(
    reinterpret_cast<Widget*>(&pool[3 * sizeof(Widget)])
);
*slot_3 = std::move(some_widget);                   // MOVE ASSIGN -- SAFE

// Clean up: manually destroy each slot
for (std::size_t i = 0; i < pool_size; ++i) {
    auto* w = std::launder(
        reinterpret_cast<Widget*>(&pool[i * sizeof(Widget)])
    );
    w->~Widget();
}
```

**Note**: The `pool` array is aligned to `Widget`'s alignment requirements. That is sufficient for all of the subsequent `Widget` elements of the pool to be aligned correctly. This is guaranteed by the C++ standard, stating that `sizeof(T)` is always a multiple of `alignof(T)` -> this is why arrays of any type work in general.

---

## 07 - Destroy-Then-Reconstruct (Replace in Place)

If you want to "replace" an object in preallocated memory with a new value without going through assignment at all, destroy first, then placement-new again:

```cpp
// Assume slot_ptr points to a live Widget in a pool
Widget* slot_ptr = /* ... */;

// Destroy the old object
slot_ptr->~Widget();

// Construct a new one in-place via move
::new (slot_ptr) Widget(std::move(new_value));      // MOVE CTOR

// The pointer must be laundered if you kept a reference
// from before the destroy (C++17 std::launder)
```

**When to prefer this over assignment:** If your type has no sensible assignment operator (e.g. const members, reference members, or non-assignable sub-objects), destroy + placement new is the only correct option.

---

## 08 - std::launder and Alignment

Two things you must not forget with raw-memory object construction:

### Alignment

```cpp
// Correct: aligned buffer (stack)
alignas(Widget) std::uint8_t buf[sizeof(Widget)];

// Also correct: aligned_alloc (C++17)
void* raw = std::aligned_alloc(alignof(Widget), sizeof(Widget));
// ...
std::free(raw); // NOT delete

// malloc is only guaranteed to align to max_align_t.
// Fine for most types, but not for over-aligned types.
```

**Note**: What are over-aligned types? Most types have an alignment requirement that's equal to their size or some natural boundary: `int` needs 4 bytes, `double` needs 8, and so on. The platform defines a "default" maximum alignment that the standard allocator guarantees, which is `__STDCPP_DEFAULT_NEW_ALIGNMENT__`, typically 16 bytes on most 64-bit systems. An over-aligned type is any type whose alignment requirement exceeds that default. You create them explicitly with `alignas`. Why would you want this? The main reasons are cache line alignment and SIMD/hardware requirements. A CPU cache line is typically 64 bytes. If you want to guarantee that a struct sits on its own cache line (to avoid false sharing between threads, for example), you align it to 64. SIMD intrinsics (SSE, AVX) require 16-byte or 32-byte alignment for their vector types.

```cpp
// Avoid false sharing: each counter on its own cache line
struct alignas(64) AtomicCounter {
    std::atomic<std::uint64_t> m_count{0};
    // Padding to fill the cache line is implicit due to alignas
};

// Thread A increments counters[0], Thread B increments counters[1].
// Without alignas(64), both might share a cache line and
// constantly invalidate each other's cache -- false sharing.
AtomicCounter counters[2];
```

### std::launder (C++17)

```cpp
// After placement new into a char/uint8_t buffer, the compiler
// may not realize a Widget now lives there. std::launder
// tells the compiler "trust me, there is a valid object here."

auto* ptr = std::launder(
    reinterpret_cast<Widget*>(buf)
);
```

---

## TL;DR - Decision Flowchart

```
Is the object already constructed at that address?
    |
    +-- NO  ->  Use placement new to call a constructor
    |             ::new (addr) T(args...)
    |             ::new (addr) T(std::forward<Args>(args)...)
    |             ::new (addr) T(std::move(src))
    |
    +-- YES ->  You have two options:
                  |
                  +-- Use assignment: *addr = src;
                  |
                  +-- Use destroy + reconstruct:
                        addr->~T();
                        ::new (addr) T(std::move(src));
                        ::new (addr) T(std::move_if_noexcept(src));
```