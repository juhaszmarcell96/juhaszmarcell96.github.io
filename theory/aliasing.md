# Aliasing

## The Aliasing Problem - `restrict` and Self-Reference

What you're remembering is the **aliasing** problem. Here's the scenario:

```cpp
void swap_bad(int& a, int& b) {
    a = a + b;
    b = a - b;
    a = a - b;
}

int x = 3;
swap_bad(x, x);  // oops - a and b refer to the same int
```

If `a` and `b` alias (refer to the same object), the function produces a completely wrong result. After `a = a + b`, both `a` and `b` are now `6` (since they're the same variable), and you end up with `x == 0`.

### Where this bites in practice

The more common and subtle case is with assignment operators:

```cpp
class String {
    char* data_;
    size_t size_;
public:
    String& operator=(const String& other) {
        delete[] data_;                    // free old data
        data_ = new char[other.size_];     // allocate new
        memcpy(data_, other.data_, other.size_); // copy
        size_ = other.size_;
        return *this;
    }
};

String s("hello");
s = s;  // self-assignment: other.data_ was just deleted!
```

When `this == &other`, you delete your own data, then try to copy from it. Use-after-free. This is why copy assignment operators need a self-assignment check:

```cpp
String& operator=(const String& other) {
    if (this == &other) return *this;  // self-assignment guard
    // ... rest of assignment
}
```

Or better, use the **copy-and-swap idiom** which handles this naturally:

```cpp
String& operator=(String other) {  // note: passed by value (copy made)
    swap(*this, other);             // swap our guts with the copy
    return *this;                   // old data destroyed when 'other' dies
}
```

This is exception-safe (the copy is made before any state is modified) and self-assignment-safe (you just swap with a copy of yourself, which is a no-op semantically).

### The compiler optimization angle

There's another dimension to aliasing that matters for performance. Consider:

```cpp
void add(int* a, int* b, int* result, int n) {
    for (int i = 0; i < n; i++) {
        result[i] = a[i] + b[i];
    }
}
```

Can the compiler vectorize this loop (use SIMD instructions)? Only if it knows `result` doesn't overlap with `a` or `b`. If `result` points into the middle of `a`, writing to `result[i]` could change what `a[i+1]` reads. The compiler must assume this is possible unless told otherwise.

In C, the `restrict` keyword tells the compiler "these pointers don't alias":

```c
void add(int* restrict a, int* restrict b, int* restrict result, int n)
```

C++ doesn't have `restrict` in the standard, but compilers support `__restrict__` as an extension. This is one reason C can sometimes produce faster numerical code than C++ out of the box - C99 has `restrict` and it's used throughout standard library math functions.

### The general principle

Any time a function takes two references or pointers of the same type, you should think: "what happens if these point to the same object?" This applies to copy assignment, move assignment, arithmetic functions, container operations, and so on.

---

## Templates: Partial Specialization and Recursive Templates

### Template basics recap

A template is a compile-time code generator. The compiler generates a separate version (instantiation) for each unique set of template arguments used.

```cpp
template<typename T>
T max(T a, T b) { return (a > b) ? a : b; }

// Compiler generates:
// int max(int, int) when you call max(3, 5)
// double max(double, double) when you call max(3.0, 5.0)
```

### Full specialization vs partial specialization

**Full specialization** provides a completely custom implementation for a specific type:

```cpp
template<typename T>
struct Hash {
    size_t operator()(const T& val) const { /* generic */ }
};

// Full specialization for std::string:
template<>
struct Hash<std::string> {
    size_t operator()(const std::string& val) const { /* string-specific */ }
};
```

**Partial specialization** specializes for a *pattern* rather than a concrete type. This is only available for class templates, not function templates:

```cpp
// Primary template:
template<typename T>
struct RemovePointer {
    using type = T;
};

// Partial specialization: if T is a pointer, strip one layer:
template<typename T>
struct RemovePointer<T*> {
    using type = T;
};

RemovePointer<int>::type      // → int (primary)
RemovePointer<int*>::type     // → int (partial spec matches)
RemovePointer<int**>::type    // → int* (strips one layer)
```

The compiler picks the most specific specialization that matches. This pattern-matching on types is the foundation of template metaprogramming.

### Recursive templates

Since partial specialization gives us pattern matching, and templates can reference other template instantiations, we get recursion. With a base case (full specialization) and a recursive case (primary or partial specialization), we can compute things at compile time.

Classic example - factorial:

```cpp
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

// Factorial<5>::value == 120, computed entirely at compile time
```

The compiler unrolls this: `Factorial<5>` instantiates `Factorial<4>`, which instantiates `Factorial<3>`, etc., until it hits the base case `Factorial<0>`.

### Type lists - the building block of `std::variant`

`std::variant` needs to store "one of these types" and know which one is active. This requires operating over a list of types at compile time. Here's how:

```cpp
// A type list is just a template that holds types:
template<typename... Ts>
struct TypeList {};

// Get the first type:
template<typename List>
struct Front;

template<typename Head, typename... Tail>
struct Front<TypeList<Head, Tail...>> {
    using type = Head;
};

// Get the size:
template<typename List>
struct Size;

template<typename... Ts>
struct Size<TypeList<Ts...>> {
    static constexpr size_t value = sizeof...(Ts);
};
```

### How `std::variant` works internally (simplified)

A variant needs three things: storage big enough for the largest type, alignment satisfying all types, and a tag (index) saying which type is active.

**The storage:**

```cpp
template<typename... Ts>
struct Variant {
    // Storage: aligned raw bytes, big enough for the largest type
    alignas(Ts...) unsigned char storage[std::max({sizeof(Ts)...})];
    
    // Tag: which type is currently stored (0, 1, 2, ...)
    size_t index;
};
```

**The interesting part is type-indexed operations** - how does `std::get<MyType>(v)` or `std::visit(visitor, v)` work? This is where recursive templates come in.

**Finding a type's index in the list:**

```cpp
// Primary: recursive case
template<typename T, typename... Ts>
struct IndexOf;

// Found it - T matches the head:
template<typename T, typename... Rest>
struct IndexOf<T, T, Rest...> {
    static constexpr size_t value = 0;
};

// Not found yet - recurse into tail:
template<typename T, typename First, typename... Rest>
struct IndexOf<T, First, Rest...> {
    static constexpr size_t value = 1 + IndexOf<T, Rest...>::value;
};

// IndexOf<double, int, float, double>::value == 2
```

**Visitation - the hardest part:**

`std::visit` takes a callable and a variant, and calls the callable with the active value. But the active type is a runtime value (the tag), and function calls need a compile-time type. The solution is a **compile-time generated jump table:**

```cpp
// Conceptually, std::visit generates something like this at compile time:
template<typename Visitor, typename... Ts>
auto visit(Visitor&& vis, Variant<Ts...>& v) {
    // Build a table of function pointers, one per type:
    using ReturnType = /* common return type */;
    using FuncPtr = ReturnType(*)(Visitor&&, void*);
    
    static constexpr FuncPtr table[] = {
        // One entry per type, generated via pack expansion:
        [](Visitor&& vis, void* storage) -> ReturnType {
            return vis(*static_cast<Ts*>(storage));
        }...
    };
    
    // Runtime dispatch via the tag:
    return table[v.index](std::forward<Visitor>(vis), &v.storage);
}
```

The compiler generates N function pointers (one per variant alternative) at compile time, stores them in an array, and does a single indexed lookup at runtime. This is O(1) dispatch - faster than a chain of `if/else` checks or `dynamic_cast`.

### Recursive template to construct/destruct the right type

Since the variant holds raw bytes, it must manually call constructors and destructors. This is done recursively:

```cpp
template<size_t I, typename... Ts>
struct VariantDestroy;

// Base case: index 0, destroy as the first type
template<typename T, typename... Rest>
struct VariantDestroy<0, T, Rest...> {
    static void destroy(void* storage) {
        static_cast<T*>(storage)->~T();
    }
};

// Recursive case: if runtime index matches I, destroy as type I
template<size_t I, typename T, typename... Rest>
struct VariantDestroy<I, T, Rest...> {
    static void destroy(size_t index, void* storage) {
        if (index == 0)
            static_cast<T*>(storage)->~T();
        else
            VariantDestroy<I - 1, Rest...>::destroy(index - 1, storage);
    }
};
```

### `if constexpr` (C++17) - the modern alternative to recursive templates

Much of the recursive template machinery can now be written more readably:

```cpp
template<typename T, typename... Ts>
constexpr size_t indexof() {
    if constexpr (sizeof...(Ts) == 0) {
        static_assert(sizeof...(Ts) > 0, "Type not found");
    } else {
        // With fold expressions and constexpr, this gets cleaner
    }
}
```

`if constexpr` evaluates at compile time and discards the untaken branch entirely - the discarded branch doesn't even need to be valid code for the given types. This replaced a huge amount of SFINAE and recursive template tricks.

### SFINAE in one paragraph

Substitution Failure Is Not An Error. When the compiler tries to instantiate a template and the substitution produces invalid code, instead of emitting an error, it just removes that overload from consideration. This is how `std::enable_if` works - it produces an invalid type to "disable" a template for certain types. C++20 concepts largely replace SFINAE with cleaner syntax: `template<typename T> requires std::integral<T>` instead of `template<typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>`.

---

Want me to cover anything else from the prep list - perhaps common concurrency patterns, lock-free data structures, or do a mock algorithmic walkthrough?