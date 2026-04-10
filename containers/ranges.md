# std::span and Ranges

How to write functions that don't care about the container.

---

## The Problem

Before C++20, if you wanted a function to accept "some contiguous sequence of ints", your options were all awkward:

```cpp
// Option 1: raw pointer + size (C-style, error-prone)
void process(const int* data, std::size_t size);

// Option 2: template (works but bloats code, hard to put in .cpp files)
template <typename Container>
void process(const Container& c);

// Option 3: pick a container (forces callers to use exactly this one)
void process(const std::vector<int>& v);

// Option 4: iterator pair (verbose, easy to mismatch)
void process(std::vector<int>::const_iterator begin,
             std::vector<int>::const_iterator end);
```

None of these cleanly say "give me a view over contiguous elements."

---

## 01 -- std::span (C++20)

`std::span<T>` is a non-owning view over a contiguous sequence of `T`. It's essentially a pointer + size pair, but type-safe. It does not allocate, does not copy, and is cheap to pass by value.

### Creating a span

```cpp
#include <span>
#include <vector>
#include <array>

std::vector<int> vec = {1, 2, 3, 4, 5};
std::array<int, 5> arr = {1, 2, 3, 4, 5};
int c_arr[] = {1, 2, 3, 4, 5};

// All of these create a std::span<int> or std::span<const int>
std::span<const int> s1(vec);               // from vector
std::span<const int> s2(arr);               // from std::array
std::span<const int> s3(c_arr);             // from C array
std::span<const int> s4(vec.data(), 3);     // from pointer + count (first 3 elements)
std::span<const int> s5(vec.begin(), 3);    // from iterator + count
std::span<const int> s6(vec.begin(),
                        vec.begin() + 3);   // from iterator pair
```

### Using span as a function parameter

This is the main use case. One function signature, accepts anything contiguous:

```cpp
// This function accepts vector, array, C array, span, string (if char), etc.
// Pass by value -- span is just a pointer + size, cheap to copy.
double average(std::span<const int> values) {
    if (values.empty()) {
        return 0.0;
    }

    double sum = 0.0;
    for (int v : values) {
        sum += v;
    }
    return sum / static_cast<double>(values.size());
}

// All of these work:
std::vector<int> vec = {1, 2, 3};
std::array<int, 4> arr = {10, 20, 30, 40};
int c_arr[] = {100, 200};

average(vec);
average(arr);
average(c_arr);
average({1, 2, 3, 4});                // from initializer list (temporary)
average(std::span(vec).subspan(1));    // elements [1, 2, 3] -> skip first
```

### const correctness

```cpp
// Read-only access: span<const T>
void read_only(std::span<const int> data);

// Mutable access: span<T>
void modify(std::span<int> data);

// span<int> implicitly converts to span<const int>, not the other way around.
// Same logic as T* converting to const T*.
```

### Static extent

`std::span` can optionally encode the size at compile time:

```cpp
// Dynamic extent (default) -- size known at runtime
std::span<int> dynamic_span;              // .size() can be anything

// Static extent -- size baked into the type
std::span<int, 5> fixed_span;            // always exactly 5 elements

// Static extent enables compile-time checks
void process_exactly_3(std::span<const int, 3> values);

std::array<int, 3> ok = {1, 2, 3};
process_exactly_3(ok);                    // compiles

std::array<int, 4> bad = {1, 2, 3, 4};
// process_exactly_3(bad);               // compile error: wrong size
```

### Subspans

```cpp
std::vector<int> vec = {10, 20, 30, 40, 50};
std::span<const int> s(vec);

auto first_3 = s.first(3);          // {10, 20, 30}
auto last_2  = s.last(2);           // {40, 50}
auto middle  = s.subspan(1, 3);     // {20, 30, 40}  (offset, count)
auto tail    = s.subspan(2);        // {30, 40, 50}  (offset to end)
```

### What span does NOT work with

`std::span` requires contiguous memory. It does not work with:

```cpp
std::list<int> lst;          // linked list -- not contiguous
std::deque<int> dq;          // deque -- not contiguous
std::set<int> st;            // tree -- not contiguous
std::unordered_map<int, int> mp;  // hash table -- not contiguous
```

For these, you need ranges.

---

## 02 -- Ranges (C++20)

Ranges generalize the idea of "something you can iterate over." Where `std::span` only works with contiguous memory, ranges work with anything that has `begin()` and `end()`.

### Range concepts

The standard provides a hierarchy of range concepts:

```
std::ranges::range                    -- has begin() and end()
    std::ranges::input_range          -- can read elements (forward)
    std::ranges::forward_range        -- multi-pass iteration
    std::ranges::bidirectional_range  -- can go backward
    std::ranges::random_access_range  -- O(1) subscript and arithmetic
    std::ranges::contiguous_range     -- elements are in contiguous memory
```

Each level adds capabilities. `std::vector` satisfies all of them. `std::list` satisfies up to `bidirectional_range`. `std::forward_list` only satisfies `forward_range`.

### Using ranges as function parameters

```cpp
#include <ranges>
#include <algorithm>
#include <concepts>

// Accept anything iterable whose elements are ints
void print_all(std::ranges::input_range auto&& r)
    requires std::same_as<std::ranges::range_value_t<decltype(r)>, int>
{
    for (const auto& elem : r) {
        std::cout << elem << " ";
    }
    std::cout << std::endl;
}

// Works with everything:
std::vector<int> vec = {1, 2, 3};
std::list<int> lst = {4, 5, 6};
std::array<int, 3> arr = {7, 8, 9};
int c_arr[] = {10, 11, 12};

print_all(vec);
print_all(lst);
print_all(arr);
print_all(c_arr);
```

### Simpler: just constrain on the concept

```cpp
// If you don't care about the element type at this level,
// just require it to be a range:
void process(std::ranges::range auto&& r) {
    for (auto&& elem : r) {
        // ...
    }
}

// Or require random access if you need subscript:
void process_indexed(std::ranges::random_access_range auto&& r) {
    for (std::size_t i = 0; i < std::ranges::size(r); ++i) {
        // r[i] is valid because random_access_range
    }
}
```

### Views and the pipe operator

Views are lazy, non-owning, composable transformations on ranges. They don't copy data -- they create a new "view" that computes elements on-the-fly during iteration.

```cpp
#include <ranges>
#include <vector>

std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

// Pipe syntax: read left to right
auto result = vec
    | std::views::filter([](int n) { return n % 2 == 0; })    // keep even
    | std::views::transform([](int n) { return n * n; })      // square them
    | std::views::take(3);                                    // first 3 results

// result is a lazy view. Nothing is computed yet.
// Iteration triggers computation:
for (int v : result) {
    std::cout << v << " ";  // prints: 4 16 36
}

// No intermediate vectors were allocated.
```

### Common views

```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};

// Filter: keep elements matching a predicate
auto evens = vec | std::views::filter([](int n) { return n % 2 == 0; });

// Transform: apply a function to each element
auto doubled = vec | std::views::transform([](int n) { return n * 2; });

// Take / drop: first N / skip first N
auto first_3 = vec | std::views::take(3);
auto skip_2  = vec | std::views::drop(2);

// Reverse: iterate backward (requires bidirectional_range)
auto rev = vec | std::views::reverse;

// Enumerate: (index, value) pairs (C++23)
for (auto [i, val] : vec | std::views::enumerate) {
    std::cout << i << ": " << val << std::endl;
}

// Iota: generate a sequence
auto zero_to_nine = std::views::iota(0, 10);

// Split / join (for strings and nested ranges)
std::string csv = "one,two,three";
auto parts = csv | std::views::split(',');

// Zip: combine two ranges element-wise (C++23)
std::vector<std::string> names = {"alice", "bob"};
std::vector<int> ages = {30, 25};
for (auto [name, age] : std::views::zip(names, ages)) {
    std::cout << name << " is " << age << "\n";
}
```

### Passing views to functions

Views are ranges too, so any function that takes a range accepts a view:

```cpp
void process(std::ranges::input_range auto&& r) {
    for (const auto& elem : r) {
        std::cout << elem << " ";
    }
    std::cout << std::endl;
}

std::vector<int> vec = {1, 2, 3, 4, 5};

process(vec);                                                    // vector directly
process(vec | std::views::filter([](int n) { return n > 2; }));  // filtered view
process(std::views::iota(0, 5));                                 // generated range
```

### Materializing a view into a container

Views are lazy. If you need to store the result, convert to a container:

```cpp
auto view = std::views::iota(1, 6)
    | std::views::transform([](int n) { return n * n; });

// C++23: ranges::to
std::vector<int> result = std::ranges::to<std::vector>(view);

// Pre-C++23: manual
std::vector<int> result2(std::ranges::begin(view), std::ranges::end(view));
```

---

## 03 -- Ranges Algorithms

C++20 also added range-based versions of all the classic `<algorithm>` functions. They take a range directly instead of iterator pairs:

```cpp
std::vector<int> vec = {5, 3, 1, 4, 2};

// Old style (iterator pairs)
std::sort(vec.begin(), vec.end());

// Range style (pass the container directly)
std::ranges::sort(vec);

// With a projection (sort by absolute value, for example)
std::vector<int> v2 = {-3, 1, -5, 2};
std::ranges::sort(v2, {}, [](int n) { return std::abs(n); });
// result: {1, 2, -3, -5}

// Find
auto it = std::ranges::find(vec, 3);

// Count
auto n = std::ranges::count_if(vec, [](int x) { return x > 2; });

// Any / all / none
bool any_negative = std::ranges::any_of(vec, [](int x) { return x < 0; });
```

The projection parameter is powerful -- it lets you sort/find/compare by a derived value without writing a custom comparator.

---

## 04 -- When to Use What

| Situation | Use |
|---|---|
| Function needs contiguous `T` elements | `std::span<const T>` |
| Function needs mutable contiguous access | `std::span<T>` |
| Function works with any iterable | `std::ranges::input_range auto&&` |
| Function needs subscript access | `std::ranges::random_access_range auto&&` |
| Lazy transformation pipeline | `std::views::filter / transform / ...` |
| Replacing iterator-pair algorithms | `std::ranges::sort / find / ...` |

The general rule: prefer `std::span` when you know the data is contiguous (it's simpler, non-templated if the element type is fixed, and has no overhead). Use ranges/concepts when you want to support non-contiguous containers or lazy pipelines.

---

## 05 -- Pitfalls and Gotchas

### span and views don't own data

```cpp
// Dangling span:
std::span<const int> bad_span() {
    std::vector<int> local = {1, 2, 3};
    return std::span(local);  // local is destroyed, span dangles
}

// Same with views:
auto bad_view() {
    std::vector<int> local = {1, 2, 3};
    return local | std::views::filter([](int n) { return n > 1; });
    // local is destroyed, view references dead memory
}
```

### View invalidation

```cpp
std::vector<int> vec = {1, 2, 3};
auto view = vec | std::views::transform([](int n) { return n * 2; });

vec.push_back(4);  // may reallocate -- view now dangles!

// Safe: don't modify the underlying container while a view exists,
// or re-create the view after modification.
```

### span from temporary

```cpp
// This is fine -- the temporary lives for the full expression:
for (int v : std::span(std::vector{1, 2, 3})) {
    // ok during iteration
}

// This dangles -- temporary is destroyed after the initialization:
std::span<const int> s = std::vector{1, 2, 3};  // dangling!
```

### Views are lazy -- side effects are surprising

```cpp
int counter = 0;
auto view = std::views::iota(0, 5)
    | std::views::transform([&counter](int n) {
        ++counter;  // side effect in a lazy view
        return n * 2;
    });

// counter is still 0 here -- nothing has been iterated yet
for (int v : view) { /* ... */ }
// counter is 5 now

// Iterating again increments counter again!
for (int v : view) { /* ... */ }
// counter is 10
```