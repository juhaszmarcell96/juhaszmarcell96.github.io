# C++ Concepts

Concepts (C++20) let you constrain templates so that invalid types produce clear errors at the point of use rather than deep inside template instantiation. They also serve as self-documenting interfaces for generic code.

---

## Standard Library Concepts

The `<concepts>` header provides foundational constraints. A few key ones:

**Type properties:** `std::integral`, `std::floating_point`, `std::signed_integral`, `std::unsigned_integral` - constrain to numeric categories.

**Comparison:** `std::equality_comparable`, `std::totally_ordered` - require that `==`, `!=`, `<`, etc. are defined.

**Object properties:** `std::movable`, `std::copyable`, `std::default_initializable`, `std::destructible` - constrain lifecycle capabilities.

**Callables:** `std::invocable<F, Args...>`, `std::predicate<F, Args...>` - require something can be called with given argument types.

**Conversions:** `std::convertible_to<From, To>`, `std::same_as<T, U>`, `std::derived_from<Derived, Base>`.

From `<iterator>` and `<ranges>` you also get `std::input_iterator`, `std::random_access_iterator`, `std::ranges::range`, `std::ranges::sized_range`, and many more.

---

## Defining Your Own Concepts

A concept is a compile-time boolean predicate on template parameters. You define one with `concept` and a constraint expression:

```cpp
template <typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;
```

You can compose concepts with `&&` and `||`, negate with `!`, or build from `requires` expressions.

---

## The `requires` Expression

A `requires` expression is a compile-time block that checks whether certain operations are valid. It comes in several forms:

### Simple requirements - "this expression must compile"

```cpp
template <typename T>
concept Addable = requires(T a, T b) {
    a + b; // just needs to be a valid expression
};
```

### Compound requirements - "this expression must compile, and its return type must satisfy..."

This is where you check return types plus qualifiers like `noexcept`:

```cpp
template <typename T>
concept Hashable = requires(T value) {
    // must compile, must be noexcept, must return std::size_t
    { std::hash<T>{}(value) } noexcept -> std::same_as<std::size_t>;
};
```

The syntax is `{ expression } noexcept -> concept<args...>;` where both `noexcept` and the return constraint are optional.

### Type requirements - "this nested type must exist"

```cpp
template <typename T>
concept HasValueType = requires {
    typename T::value_type;
};
```

### Nested requirements - "this compile-time predicate must be true"

```cpp
template <typename T>
concept SmallObject = requires {
    requires sizeof(T) <= 64;
};
```

Note the inner `requires` keyword - without it you'd just be checking that `sizeof(T) <= 64` is a valid expression (which it always is), not that it evaluates to `true`.

---

## Building Interface-Like Concepts

This is where concepts become powerful - you can define something analogous to an interface or trait, requiring specific member types, functions, const-correctness, and noexcept guarantees. This `TransitionExecutor` example does exactly this:

```cpp
template <typename TransitionExecutorType>
concept TransitionExecutor =
    // type requirement: must have a nested state_type
    requires { typename TransitionExecutorType::state_type; } &&

    // compound requirement: must have a transition() member
    requires (TransitionExecutorType executor,
              typename TransitionExecutorType::state_type from,
              typename TransitionExecutorType::state_type to)
    {
        { executor.transition(from, to) } noexcept -> std::same_as<bool>;
    };
```

What this checks: the type must expose a `state_type` alias, and it must have a `transition` method that takes two `state_type` values, is `noexcept`, and returns `bool`.

Here is a more elaborate example showing how to require const member functions, mutable member functions, static members, and data access:

```cpp
template <typename T>
concept Serializable =
    // must have these nested types
    requires { typename T::format_type; } &&

    requires (T obj, const T const_obj, typename T::format_type fmt) {
        // const member function, returns a std::string
        { const_obj.serialize(fmt) } noexcept -> std::same_as<std::string>;

        // mutable member function, returns bool
        { obj.deserialize(std::declval<std::string_view>(), fmt) } -> std::same_as<bool>;

        // static member function
        { T::default_format() } noexcept -> std::same_as<typename T::format_type>;

        // must be nothrow destructible
        requires std::is_nothrow_destructible_v<T>;
    };
```

The trick for checking const-correctness: introduce a `const T` parameter in the `requires` clause. If you call a method on `const_obj`, it will only compile if that method is declared `const`. Similarly, if you want to ensure something is *not* const-qualified, call it on the non-const parameter.

---

## Using Concepts in Templates

There are four equivalent syntactic forms:

### 1. Constrained template parameter (most concise)

```cpp
template <TransitionExecutor T>
class StateMachine {
    // ...
};
```

### 2. `requires` clause after the template parameter list

```cpp
template <typename T>
requires TransitionExecutor<T>
class StateMachine {
    // ...
};
```

### 3. Trailing `requires` clause (works on functions)

```cpp
template <typename T>
bool run_transition(T& executor,
                    typename T::state_type from,
                    typename T::state_type to)
    requires TransitionExecutor<T>
{
    return executor.transition(from, to);
}
```

### 4. Abbreviated function template with `auto`

```cpp
void process(TransitionExecutor auto& executor) {
    // TransitionExecutor is checked on whatever type is deduced
}
```

All four produce the same constraint; choose based on readability. The constrained parameter form is cleanest when there is one concept per parameter. The `requires` clause is better when you need compound conditions:

```cpp
template <typename T>
requires TransitionExecutor<T> && std::copyable<T>
class RedundantStateMachine {
    // ...
};
```

---

## Concept Subsumption (Overload Resolution)

When multiple overloads are constrained, the compiler picks the most specific one. If concept A implies concept B (A subsumes B), then a function constrained on A is preferred:

```cpp
template <typename T>
concept Animal = requires(T a) {
    { a.speak() } -> std::same_as<std::string>;
};

template <typename T>
concept Pet = Animal<T> && requires(T a) {
    { a.name() } -> std::same_as<std::string>;
};

// if T satisfies Pet, this overload wins because Pet subsumes Animal
void greet(Pet auto& p) { /* ... */ }
void greet(Animal auto& a) { /* ... */ }
```

This only works through direct syntactic subsumption - the compiler checks the logical structure of the concept definitions, so compose concepts explicitly with `&&` to enable this.