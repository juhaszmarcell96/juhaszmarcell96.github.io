# Templates

## Template basics recap

A template is a compile-time code generator. The compiler generates a separate version (instantiation) for each unique set of template arguments used.

```cpp
template<typename T>
T max(T a, T b) { return (a > b) ? a : b; }

// Compiler generates:
// int max(int, int) when you call max(3, 5)
// double max(double, double) when you call max(3.0, 5.0)
```

## Full specialization vs partial specialization

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

## Recursive templates

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

## Type lists - the building block of `std::variant`

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