# std::launder

`std::launder` takes a pointer and returns a pointer to the same address, but it tells the compiler: "forget everything you assumed about what lives at this address - there is a valid object here, trust me." It's a compiler optimization barrier for pointer provenance.

The signature is deceptively simple:

```cpp
template <class T>
constexpr T* std::launder(T* ptr) noexcept;
```

At runtime it's a no-op, it returns the same address. The effect is purely at the compilation/optimization level.

## Why it exists

The C++ standard gives the compiler permission to assume that certain properties of an object don't change during its lifetime. When you violate those assumptions (by ending one object's lifetime and starting another's at the same address), the compiler might optimize based on stale knowledge. `std::launder` is the mechanism to say "your assumptions are invalidated, re-read from this address."

**Note**: `std::launder` and `volatile` are both about preventing the compiler from making assumptions, but they solve completely different problems and operate at different levels. `volatile` tells the compiler: "the value at this address can change at any time due to something outside the program's control - don't cache reads, don't elide writes, don't reorder accesses to this variable." The classic use cases are memory-mapped hardware registers, signal handlers, and (in the old days, incorrectly) threading. It affects every single read and write to that variable for its entire lifetime. `std::launder` tells the compiler: "your pointer provenance tracking is wrong - there is a valid object at this address that you don't know about." It's a one-time correction to the compiler's internal model of what lives where. After the launder call, the compiler can go right back to optimizing normally based on the new object's properties.

## The three scenarios where you need it

### 1. After placement new into a typed buffer

When you destroy an object and construct a new one at the same address, the compiler may still assume the old object is there - especially if it had `const` or reference members:

```cpp
struct Config {
    const std::uint32_t m_id;
    double m_value;
};

alignas(Config) std::uint8_t buf[sizeof(Config)];

// Construct first object
auto* cfg = ::new (buf) Config{42, 3.14};

std::cout << cfg->m_id << std::endl;   // 42

// Destroy and reconstruct with different const value
cfg->~Config();
auto* cfg2 = ::new (buf) Config{99, 2.71};

// Problem: the compiler saw cfg->m_id was const 42.
// It may constant-fold and still return 42 here.
std::cout << cfg->m_id << std::endl;   // might print 42 due to optimization!

// Solution: launder the pointer
auto* cfg3 = std::launder(reinterpret_cast<Config*>(buf));
std::cout << cfg3->m_id << std::endl;  // guaranteed to print 99
```

The compiler is allowed to assume a `const` member never changes. Since `m_id` was `42` when the pointer `cfg` was created, the compiler can cache that. `std::launder` breaks that assumption.

### 2. Accessing objects through reinterpret_cast from a byte buffer

When you placement-new into a `char` / `std::uint8_t` / `std::byte` array and then access the object through a reinterpreted pointer, the compiler doesn't necessarily know an object of your type lives there:

```cpp
alignas(Widget) std::uint8_t pool[sizeof(Widget)];
::new (pool) Widget();

// The compiler knows 'pool' as a uint8_t array.
// reinterpret_cast alone does not tell it a Widget is there.
auto* w = std::launder(reinterpret_cast<Widget*>(pool));
w->do_something();   // safe
```

Note: if you capture the return value of placement new directly, you already have a valid pointer and don't need `std::launder`:

```cpp
// This is fine without launder -- placement new returns a Widget*
Widget* w = ::new (pool) Widget();
w->do_something();   // safe, no launder needed
```

You only need `std::launder` when you *didn't* keep placement new's return value and you're reconstructing the pointer from the underlying storage.

### 3. After replacing an object with const or reference members (even without raw memory)

This can happen with `std::optional`-like types or union-based storage internally:

```cpp
struct Entry {
    const std::string m_key;
    int m_value;
};

// Inside some container implementation that uses a union:
union Storage {
    Entry m_entry;
    Storage() {}
    ~Storage() {}
};

Storage s;
::new (&s.m_entry) Entry{"hello", 1};
s.m_entry.~Entry();
::new (&s.m_entry) Entry{"world", 2};

// Without launder, compiler may assume m_key is still "hello"
auto* e = std::launder(&s.m_entry);
std::cout << e->m_key << std::endl;   // guaranteed "world"
```

## When you do NOT need it

You don't need `std::launder` when you keep the pointer returned by placement new (it's already valid), when you're working with objects that have no `const` members, no reference members, and no change of dynamic type at that address, or when using `std::allocator` and standard containers (they handle this internally).

## Mental model

Think of it as: "the compiler tracks what it thinks lives at each address. When you do something behind its back (destroy + reconstruct, especially with const/reference members), the compiler's model becomes stale. `std::launder` forces it to look again."

The key points are:
- it's a compile-time optimization barrier for pointer provenance
- it's needed after placement new when you access through a pointer that wasn't returned by that placement new
- it's especially critical when `const` or reference members are involved because the compiler can constant-fold those
- it's a no-op at runtime - the entire purpose is to prevent the optimizer from using stale assumptions