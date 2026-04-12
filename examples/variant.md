```cpp
// simple_variant.cpp
// A minimal but functional variant implementation for interview study.
// Focuses on: aligned storage, type-indexed access, recursive templates.
// Compile: g++ -std=c++17 -Wall -o variant simple_variant.cpp && ./variant

#include <cstddef>
#include <iostream>
#include <new>       // placement new
#include <string>
#include <type_traits>
#include <utility>

// ============================================================================
// HELPER: Find the maximum of a pack of values at compile time
// ============================================================================
template<size_t First, size_t... Rest>
struct MaxOf {
    static constexpr size_t value =
        First > MaxOf<Rest...>::value ? First : MaxOf<Rest...>::value;
};

template<size_t Only>
struct MaxOf<Only> {
    static constexpr size_t value = Only;
};

// ============================================================================
// HELPER: Find the index of type T in a parameter pack
// Uses partial specialization + recursion.
// ============================================================================

// Primary template: recursive case
template<typename T, typename... Ts>
struct IndexOf;

// Match found: T is the head of the list → index is 0
template<typename T, typename... Rest>
struct IndexOf<T, T, Rest...> {
    static constexpr size_t value = 0;
};

// No match yet: strip the head, recurse, add 1
template<typename T, typename First, typename... Rest>
struct IndexOf<T, First, Rest...> {
    static constexpr size_t value = 1 + IndexOf<T, Rest...>::value;
};

// ============================================================================
// HELPER: Get the Nth type from a parameter pack
// ============================================================================

// Primary template: recursive case — peel off one type, decrement N
template<size_t N, typename... Ts>
struct NthType;

template<size_t N, typename First, typename... Rest>
struct NthType<N, First, Rest...> {
    using type = typename NthType<N - 1, Rest...>::type;
};

// Base case: N == 0, the head is our type
template<typename First, typename... Rest>
struct NthType<0, First, Rest...> {
    using type = First;
};

// Convenience alias
template<size_t N, typename... Ts>
using NthType_t = typename NthType<N, Ts...>::type;

// ============================================================================
// HELPER: Destroy the active object in raw storage.
// We need to call the right destructor based on a runtime index.
// This generates a compile-time table of destructor calls.
// ============================================================================

template<typename... Ts>
struct VariantDestroyer;

// Base case: no types left, nothing to destroy (shouldn't be reached)
template<>
struct VariantDestroyer<> {
    static void destroy(size_t /*index*/, void* /*storage*/) {}
};

// Recursive case: if index == 0, destroy as Head; otherwise recurse
template<typename Head, typename... Tail>
struct VariantDestroyer<Head, Tail...> {
    static void destroy(size_t index, void* storage) {
        if (index == 0) {
            static_cast<Head*>(storage)->~Head();
        } else {
            VariantDestroyer<Tail...>::destroy(index - 1, storage);
        }
    }
};

// ============================================================================
// HELPER: Copy-construct the active object into new storage.
// Same recursive dispatch pattern as destroyer.
// ============================================================================

template<typename... Ts>
struct VariantCopier;

template<>
struct VariantCopier<> {
    static void copy(size_t, const void*, void*) {}
};

template<typename Head, typename... Tail>
struct VariantCopier<Head, Tail...> {
    static void copy(size_t index, const void* src, void* dst) {
        if (index == 0) {
            // Placement new: construct a Head in dst, copying from src
            new (dst) Head(*static_cast<const Head*>(src));
        } else {
            VariantCopier<Tail...>::copy(index - 1, src, dst);
        }
    }
};

// ============================================================================
// THE VARIANT ITSELF
// ============================================================================

template<typename... Ts>
class Variant {
    // --- Storage ---
    // We need a buffer that is:
    //   - large enough to hold the largest type
    //   - aligned to the strictest alignment of all types
    //
    // alignas(Ts...) expands to alignas(T1, T2, T3, ...) which picks
    // the strictest alignment. MaxOf computes the largest sizeof.

    static constexpr size_t StorageSize = MaxOf<sizeof(Ts)...>::value;

    alignas(Ts...) unsigned char storage_[StorageSize];

    // --- Type tag ---
    // Which type is currently active. Matches the position in Ts...
    // npos means "valueless by exception" (no active object)
    size_t index_;

    static constexpr size_t npos = static_cast<size_t>(-1);

    // --- Internal helpers ---

    void destroy_current() {
        if (index_ != npos) {
            VariantDestroyer<Ts...>::destroy(index_, &storage_);
            index_ = npos;
        }
    }

public:
    // --- Default constructor: constructs the first type ---

    Variant() : index_(0) {
        using First = NthType_t<0, Ts...>;
        new (&storage_) First();
    }

    // --- Constructing from a value ---
    // This constructor is enabled only if T is one of the variant's types.
    // We use SFINAE via enable_if to prevent ambiguity.

    template<
        typename T,
        // Remove cv-ref so that e.g. "const string&" matches "string"
        typename Decayed = std::decay_t<T>,
        // Only enable if Decayed is one of our types
        typename = std::enable_if_t<(std::is_same_v<Decayed, Ts> || ...)>
    >
    Variant(T&& value) : index_(IndexOf<Decayed, Ts...>::value) {
        new (&storage_) Decayed(std::forward<T>(value));
    }

    // --- Copy constructor ---

    Variant(const Variant& other) : index_(other.index_) {
        if (index_ != npos) {
            VariantCopier<Ts...>::copy(index_, &other.storage_, &storage_);
        }
    }

    // --- Move constructor ---

    // We'd need a VariantMover helper (same pattern as Copier).
    // Omitted for brevity — the pattern is identical but calls
    // new (dst) Head(std::move(*static_cast<Head*>(src)))

    // --- Destructor ---

    ~Variant() {
        destroy_current();
    }

    // --- Assignment from a value ---

    template<
        typename T,
        typename Decayed = std::decay_t<T>,
        typename = std::enable_if_t<(std::is_same_v<Decayed, Ts> || ...)>
    >
    Variant& operator=(T&& value) {
        destroy_current();
        new (&storage_) Decayed(std::forward<T>(value));
        index_ = IndexOf<Decayed, Ts...>::value;
        return *this;
    }

    // --- Copy assignment ---

    Variant& operator=(const Variant& other) {
        if (this != &other) {               // self-assignment check!
            destroy_current();
            index_ = other.index_;
            if (index_ != npos) {
                VariantCopier<Ts...>::copy(index_, &other.storage_, &storage_);
            }
        }
        return *this;
    }

    // --- Accessors ---

    // index() returns which type is active
    size_t index() const { return index_; }

    // holds_alternative<T>(): is T the active type?
    template<typename T>
    bool holds_alternative() const {
        return index_ == IndexOf<T, Ts...>::value;
    }

    // get<T>(): access the stored value by type.
    // Undefined behavior if T is not the active type (real std::variant
    // throws std::bad_variant_access; we keep it simple here).
    template<typename T>
    T& get() {
        // In production you'd check and throw. For our purposes:
        return *static_cast<T*>(static_cast<void*>(&storage_));
    }

    template<typename T>
    const T& get() const {
        return *static_cast<const T*>(static_cast<const void*>(&storage_));
    }

    // get<I>(): access the stored value by index
    template<size_t I>
    NthType_t<I, Ts...>& get() {
        return *static_cast<NthType_t<I, Ts...>*>(static_cast<void*>(&storage_));
    }

    template<size_t I>
    const NthType_t<I, Ts...>& get() const {
        return *static_cast<const NthType_t<I, Ts...>*>(static_cast<const void*>(&storage_));
    }

    // --- Visit ---
    // Calls visitor(active_value). Uses the same recursive dispatch pattern.
    // Real std::visit uses a function pointer table for O(1) dispatch;
    // this recursive version is O(n) in the number of types but clearer.

    template<typename Visitor>
    auto visit(Visitor&& vis) {
        return visit_impl<Visitor, Ts...>(std::forward<Visitor>(vis), 0);
    }

private:
    // Recursive visit: try each type in order
    template<typename Visitor, typename Head, typename... Tail>
    auto visit_impl(Visitor&& vis, size_t current_index) {
        if (index_ == current_index) {
            return vis(*static_cast<Head*>(static_cast<void*>(&storage_)));
        }
        if constexpr (sizeof...(Tail) > 0) {
            return visit_impl<Visitor, Tail...>(
                std::forward<Visitor>(vis), current_index + 1);
        }
        else {
            // Should never reach here if variant is valid
            __builtin_unreachable();
        }
    }
};

// ============================================================================
// DEMO
// ============================================================================

int main() {
    std::cout << "=== Storage layout ===\n";
    std::cout << "Variant<int, double, std::string> size: "
              << sizeof(Variant<int, double, std::string>) << " bytes\n";
    std::cout << "  (int: " << sizeof(int)
              << ", double: " << sizeof(double)
              << ", string: " << sizeof(std::string)
              << ", + index tag)\n\n";

    // Construct with an int
    Variant<int, double, std::string> v(42);
    std::cout << "=== After constructing with int 42 ===\n";
    std::cout << "index: " << v.index() << "\n";
    std::cout << "holds int? " << v.holds_alternative<int>() << "\n";
    std::cout << "value: " << v.get<int>() << "\n\n";

    // Assign a string
    v = std::string("hello variant");
    std::cout << "=== After assigning string ===\n";
    std::cout << "index: " << v.index() << "\n";
    std::cout << "holds string? " << v.holds_alternative<std::string>() << "\n";
    std::cout << "value: " << v.get<std::string>() << "\n\n";

    // Assign a double
    v = 3.14;
    std::cout << "=== After assigning double ===\n";
    std::cout << "index: " << v.index() << "\n";
    std::cout << "get<1>(): " << v.get<1>() << "\n\n";

    // Visit
    v = 99;
    std::cout << "=== Visit ===\n";
    v.visit([](auto& val) {
        std::cout << "Visited value: " << val
                  << " (type size: " << sizeof(val) << ")\n";
    });

    // Copy
    Variant<int, double, std::string> v2 = v;
    std::cout << "\n=== After copy ===\n";
    std::cout << "v2 holds int? " << v2.holds_alternative<int>() << "\n";
    std::cout << "v2 value: " << v2.get<int>() << "\n";

    // Reassign original — copy is independent
    v = std::string("modified");
    std::cout << "v2 still: " << v2.get<int>() << "\n";
    std::cout << "v now: " << v.get<std::string>() << "\n";

    return 0;
}
```