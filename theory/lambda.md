# C++ Lambdas

A lambda is syntactic sugar for an anonymous callable object. The compiler generates a unique, unnamed struct with an `operator()` - conceptually identical to what you would write by hand as a functor, but far more concise.

---

## Syntax Breakdown

```cpp
[capture](parameters) mutable constexpr noexcept -> return_type { body }
```

Every part except the capture list and the body is optional. A minimal lambda:

```cpp
auto f = []{ return 42; };
```

### Capture list `[...]`

Controls which variables from the enclosing scope the lambda can access, and how.

`[]` - capture nothing. The lambda can only use its own parameters and globals.

`[x]` - capture `x` by value. A copy is made when the lambda is created. The copy is `const` by default inside the lambda body.

`[&x]` - capture `x` by reference. The lambda sees the original variable, including mutations by either side. The original must outlive the lambda.

`[=]` - capture everything used in the body by value (default copy).

`[&]` - capture everything used in the body by reference (default reference).

`[=, &x]` - capture everything by value, except `x` by reference. You can mix defaults with explicit overrides.

`[&, x]` - capture everything by reference, except `x` by value.

`[x = std::move(some_obj)]` - **init capture** (C++14). Creates a new capture variable initialized by an arbitrary expression. Essential for move-only types:

```cpp
auto ptr = std::make_unique<Widget>();
auto f = [p = std::move(ptr)]{ p->do_something(); };
```

`[...args = std::move(args)]` - **pack init capture** (C++20). Moves a parameter pack into the lambda:

```cpp
template <typename... Args>
auto bind_all(Args&&... args) {
    return [...captured = std::forward<Args>(args)]{ /* use captured... */ };
}
```

### `this` capture

`[this]` - captures the `this` pointer by value (the pointer itself, not the object). The lambda can access all members of the enclosing class, but the object must outlive the lambda. This is a common source of dangling-pointer bugs with async code.

`[*this]` - captures the entire object by value (C++17). Makes a copy, so the lambda is self-contained and safe to outlive the object:

```cpp
class Processor {
    std::int32_t m_value = 10;
public:
    auto make_callback() const {
        // safe: lambda owns a copy of the Processor
        return [*this]{ return m_value; };
    }
};
```

`[=]` inside a member function implicitly captures `this` (the pointer). C++20 deprecated this implicit capture - prefer being explicit with `[this]` or `[*this]`.

### Parameters `(...)`

Same as regular function parameters. Since C++14, you can use `auto` for generic lambdas:

```cpp
auto add = [](auto a, auto b){ return a + b; };
```

Since C++20, you can use explicit template syntax:

```cpp
auto add = []<typename T>(T a, T b){ return a + b; };
```

This is useful when you need to name the type, constrain it with a concept, or use it inside the body.

### `mutable`

By default, a lambda's `operator()` is `const`, meaning value-captured variables cannot be modified. `mutable` removes that const:

```cpp
auto counter = [n = 0]() mutable { return ++n; };
counter(); // 1
counter(); // 2
```

### `constexpr` and `consteval`

`constexpr` (implicit since C++17 when possible) allows the lambda to be evaluated at compile time. `consteval` (C++20) forces compile-time-only evaluation.

### `noexcept`

Declares that the lambda will not throw. Same semantics as `noexcept` on any other function - `std::terminate` is called if an exception escapes.

### Return type `-> T`

Usually deduced automatically. Specify it explicitly when the body has multiple return paths with different types, or when you want to enforce a conversion:

```cpp
auto f = [](std::int32_t x) -> double { return x; };
```

---

## What the Compiler Actually Generates

When you write:

```cpp
std::int32_t offset = 10;
auto add_offset = [offset](std::int32_t x) noexcept -> std::int32_t {
    return x + offset;
};
```

The compiler generates something equivalent to:

```cpp
class __anonymous_lambda_42 {
    std::int32_t m_offset;
public:
    __anonymous_lambda_42(std::int32_t offset) noexcept : m_offset{offset} {}

    [[nodiscard]] std::int32_t operator()(std::int32_t x) const noexcept {
        return x + m_offset;
    }
};
```

Key observations: each lambda expression produces a **unique type**, even if two lambdas are textually identical. Captured-by-value variables become member fields. Captured-by-reference variables become reference members. The `operator()` is `const` unless you specify `mutable`. Generic lambdas (`auto` parameters) produce a templated `operator()`.

### Captureless lambdas convert to function pointers

A lambda that captures nothing has a special property - it can implicitly convert to a plain function pointer:

```cpp
void register_callback(void (*fn)(std::int32_t));

register_callback([](std::int32_t x){ /* ... */ }); // works
```

This works because with no state to carry, the compiler can emit a regular free function. The moment you capture anything, this conversion is no longer possible.

---

## `std::function` vs Templates vs Function Pointers

Since each lambda has a unique anonymous type, you cannot spell it directly. There are several ways to pass lambdas around:

**Template parameter** - zero overhead, the compiler inlines everything. Preferred in most cases:

```cpp
template <typename Func>
void apply(std::int32_t value, Func func) {
    func(value);
}
```

**`std::function`** - type-erased wrapper. Carries overhead (heap allocation for large captures, virtual dispatch). Use when you need to store heterogeneous callables in a container or as a member variable:

```cpp
std::function<void(std::int32_t)> m_callback;
```

**Function pointer** - only works for captureless lambdas. Useful for C API interop.

**`auto` return / parameter** - when you can let the type flow through without erasing it:

```cpp
[[nodiscard]] auto make_adder(std::int32_t offset) noexcept {
    return [offset](std::int32_t x) noexcept { return x + offset; };
}
```

---

## Common Patterns

### Immediately invoked lambda (IIFE)

Useful for complex initialization of `const` variables:

```cpp
const auto config = [&] {
    Config c;
    c.m_port = read_env("PORT");
    c.m_host = read_env("HOST");
    // ... more complex setup
    return c;
}(); // note the () -- invoked immediately
```

Without this pattern, `config` could not be `const` because you would need to mutate it across multiple statements.

### Lambda as comparator

```cpp
std::sort(data.begin(), data.end(),
    [](const Widget& a, const Widget& b) noexcept
    {
        return a.priority() < b.priority();
    });
```

### Recursive lambda

A lambda cannot directly call itself because the `auto` variable is not yet initialized when the lambda body is parsed. The classic workaround:

```cpp
// pass itself as a parameter
auto factorial = [](auto self, std::int64_t n) -> std::int64_t {
    if (n <= 1) { return 1; }
    return n * self(self, n - 1);
};
auto result = factorial(factorial, 10);
```

C++23 introduces `std::move_only_function` and deducing `this`, which enable cleaner recursive lambdas. In C++23, the **explicit object parameter** feature lets any member function (including `operator()`) declare its first parameter as `this auto& self` (or any variation). Instead of the implicit `this` pointer that member functions normally receive behind the scenes, you get an explicitly named, deduced parameter that is the object the function is called on.

```cpp
// C++23 deducing this
auto factorial = [](this auto& self, std::int64_t n) -> std::int64_t {
    if (n <= 1) { return 1; }
    return n * self(n - 1);
};
auto result = factorial(10); // no need to explicitly pass the lambda as parameter
```

---

## Lifetime Pitfalls

The single most important thing to watch for: **reference captures create dangling references if the lambda outlives the captured variables.** This is especially dangerous when lambdas are stored for later execution - callbacks, async tasks, posted to event loops:

```cpp
std::function<std::int32_t()> make_bad_lambda() {
    std::int32_t local = 42;
    return [&local]{ return local; }; // dangling reference
}
```

The rule of thumb: if the lambda escapes the current scope, capture by value or use init captures with `std::move`. Reserve reference captures for lambdas that are consumed synchronously within the same scope, such as those passed to `std::sort` or `std::for_each`.