# Any

**IMPORTANT**: deleting `void*` is undefined behavior

If you have an object and a `void*` pointer to that object, you cannot simply call `delete` on the pointer, you have to first cast it, maybe even `std::launder` it.

```cpp
#include <typeinfo>
#include <utility>
#include <stdexcept>
#include <cstddef>

class BadAnyCast : public std::bad_cast {
public:
    const char* what() const noexcept override {
        return "bad any_cast";
    }
};

class Any {
private:
    // -- Operations table: replaces the vtable --
    struct Ops {
        void (*destroy)(void*);            // destroy
        void* (*clone)(const void*);       // clone
        const std::type_info& (*type)();   // type
    };

    // One static instance per type, generated at compile time
    template <typename T>
    static constexpr Ops ops_for = {
        [](void* ptr) { delete static_cast<T*>(ptr); },
        [](const void* ptr) -> void* { return new T(*static_cast<const T*>(ptr)); },
        []() -> const std::type_info& { return typeid(T); }
    };

    void* m_storage;
    const Ops* m_ops;

    template <typename T>
    friend T any_cast(const Any&);

    template <typename T>
    friend T any_cast(Any&);

    template <typename T>
    friend T any_cast(Any&&);

    template <typename T>
    friend const T* any_cast(const Any*) noexcept;

    template <typename T>
    friend T* any_cast(Any*) noexcept;
public:
    Any() noexcept : m_storage(nullptr), m_ops(nullptr) {}

    template <typename T, typename = std::enable_if_t<!std::is_same_v<std::decay_t<T>, Any>>>
    Any(T&& value)
        : m_storage(new std::decay_t<T>(std::forward<T>(value)))
        , m_ops(&ops_for<std::decay_t<T>>) {}

    Any(const Any& other)
        : m_storage(other.m_ops ? other.m_ops->clone(other.m_storage) : nullptr)
        , m_ops(other.m_ops) {}

    Any(Any&& other) noexcept
        : m_storage(std::exchange(other.m_storage, nullptr))
        , m_ops(std::exchange(other.m_ops, nullptr)) {}

    ~Any() {
        reset();
    }

    Any& operator=(const Any& other) {
        if (this != &other) {
            Any tmp(other);
            swap(tmp);
        }
        return *this;
    }

    Any& operator=(Any&& other) noexcept {
        if (this != &other) {
            reset();
            m_storage = std::exchange(other.m_storage, nullptr);
            m_ops = std::exchange(other.m_ops, nullptr);
        }
        return *this;
    }

    template <typename T,
              typename = std::enable_if_t<!std::is_same_v<std::decay_t<T>, Any>>>
    Any& operator=(T&& value) {
        Any tmp(std::forward<T>(value));
        swap(tmp);
        return *this;
    }

    [[nodiscard]] bool has_value() const noexcept {
        return m_storage != nullptr;
    }

    [[nodiscard]] const std::type_info& type() const noexcept {
        return m_ops ? m_ops->type() : typeid(void);
    }

    void reset() noexcept {
        if (m_ops) {
            m_ops->destroy(m_storage);
            m_storage = nullptr;
            m_ops = nullptr;
        }
    }

    void swap(Any& other) noexcept {
        std::swap(m_storage, other.m_storage);
        std::swap(m_ops, other.m_ops);
    }

    template <typename T, typename... Args>
    std::decay_t<T>& emplace(Args&&... args) {
        reset();
        auto* ptr = new std::decay_t<T>(std::forward<Args>(args)...);
        m_storage = ptr;
        m_ops = &ops_for<std::decay_t<T>>;
        return *ptr;
    }
};

// -- any_cast implementations --

template <typename T>
T any_cast(const Any& a) {
    using U = std::remove_cvref_t<T>;
    if (!a.m_ops || a.m_ops->type() != typeid(U)) {
        throw BadAnyCast();
    }
    return *static_cast<const U*>(a.m_storage);
}

template <typename T>
T any_cast(Any& a) {
    using U = std::remove_cvref_t<T>;
    if (!a.m_ops || a.m_ops->type() != typeid(U)) {
        throw BadAnyCast();
    }
    return *static_cast<U*>(a.m_storage);
}

template <typename T>
T any_cast(Any&& a) {
    using U = std::remove_cvref_t<T>;
    if (!a.m_ops || a.m_ops->type() != typeid(U)) {
        throw BadAnyCast();
    }
    return std::move(*static_cast<U*>(a.m_storage));
}

template <typename T>
const T* any_cast(const Any* a) noexcept {
    if (a && a->m_ops && a->m_ops->type() == typeid(T)) {
        return static_cast<const T*>(a->m_storage);
    }
    return nullptr;
}

template <typename T>
T* any_cast(Any* a) noexcept {
    if (a && a->m_ops && a->m_ops->type() == typeid(T)) {
        return static_cast<T*>(a->m_storage);
    }
    return nullptr;
}
```

**Key differences from the virtual version:**

The `Ops` struct is the manual vtable - three function pointers for destroy, clone, and type. The template variable `ops_for<T>` generates one static `Ops` instance per type at compile time, with lambdas that know how to operate on `T`. The stored value is just a `void*` now, so `Any` itself is two pointers wide (data + ops) rather than one pointer to a heap-allocated polymorphic object.

**Why this can be better:**

No inheritance hierarchy means no `Placeholder` / `Holder<T>` - you allocate just the `T` itself, not a `Holder<T>` that wraps it. The function pointer calls are indirect (like virtual), but the compiler has a better chance of devirtualizing them since the ops pointer is `constexpr` and set from a known static. It also makes adding SBO more natural - you'd replace `void* m_storage` with a `union` of a pointer and an aligned buffer, and add a fourth function pointer for "move into buffer," without needing to restructure an inheritance hierarchy.

This is essentially what most real `std::any` implementations (libstdc++, libc++) actually do - function-pointer tables over `void*` storage with SBO, not virtual dispatch.

## `enable_if_t` in the constructor

```cpp
template <typename T, typename = std::enable_if_t<!std::is_same_v<std::decay_t<T>, Any>>>
```

The second `typename` is an optional template parameter with no name and a default value. If `!std::is_same_v<std::decay_t<T>, Any>` evaluates to `true`, meaning that the decayed type is not `Any`, then the `std::enable_if` has a public `type` typedef. The type of this typedef is the second argument of `std::enable_if`, which has a default value `void`. So in this case, if `T` is not `Any`, then we have `template <typename T, typename = void>`, otherwise this is a failed substitution. And according to SFINAE (Substitution Failure Is Not An Error), the compiler will discard this substitution. This forces the compiler to use the copy or move constructors when with `Any` types.

`std::decay` is needed, because without it, if someone passes `Any&` or `const Any&`, `std::is_same_v<T, Any>` would be `false` (because `T` deduces as `Any&`), and the template constructor would hijack the call instead of the copy constructor. `decay_t` strips references and cv-qualifiers so the check works regardless of how `Any` is passed.

## How it works

```cpp
struct S {};
Any a { S{} };
```

The `Any` class defines a static instance `ops_for` for the given type. `ops_for` is not instantiated when `Any` is defined, it is rather instantiated *per type* when first used. The function pointers of this static are set like this:

```cpp
void destroy (void* ptr) {
    delete static_cast<T*>(ptr);
}
void* clone (const void* ptr) {
    return new T(*static_cast<const T*>(ptr));
}
const std::type_info& type () {
    return typeid(T);
}
```

`m_ops` points to the static `ops_for` instance that has the correctly typed function pointers and acts as a `vtable`. `m_storage` contains the actual value that the function operate on. So, when the destructor calls `m_ops->destroy(m_storage)`, then the static instance's `destroy` function will cast `m_storage` to the correct `T` and call `delete` on it.

## casting

The any_cast uses `typeid` comparison, which means it checks exact type identity - not inheritance. If you store a `Derived*` and try to `any_cast<Base*>`, it will throw even though the conversion would be valid in normal C++. This is a deliberate design choice in `std::any` (and this implementation mirrors it). The reason is that without a virtual hierarchy on the stored type, there's no safe way to do a polymorphic cast from a `void*`.

## launder

Why is `std::launder` not needed here?

Good question. The short answer is: in this implementation, `std::launder` is not needed, but the reason isn't that the compiler "knows" what's there - it's about how the pointer was obtained.

When you do placement new into a `char[]` buffer, the issue is that the compiler originally associated that memory with `char` objects. The pointer you get back from placement new is valid, but if you later access the object through the original `char*` (cast to `T*`), the compiler's aliasing and object-lifetime reasoning can go wrong - it doesn't know a `T` was constructed there. `std::launder` tells the compiler "forget what you think lives at this address and treat it as pointing to the object that's actually there now."

In the `void*` / heap approach here, the situation is different. `new T(value)` both allocates and constructs, and returns a `T*`. That `T*` is immediately stored as `void*`. When you later cast it back with `static_cast<T*>(m_storage)`, you're recovering the original pointer value that `new` returned - the one that was always associated with a `T` object. There's no reuse of storage, no construction of a different type in existing memory. The round-trip `T*` -> `void*` -> `T*` via `static_cast` is well-defined and gives you back a pointer to the same `T` object.

So the distinction is:

- **Placement new into `char[]`**: the memory has a prior identity, you construct a new object over it, the original pointer wasn't a `T*` -> need `std::launder`
- **`new T` stored as `void*`**: the pointer originated as `T*`, you're just round-tripping through `void*` -> no `std::launder` needed