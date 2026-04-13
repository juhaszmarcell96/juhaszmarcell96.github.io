# Exceptions

## Throwing Exceptions

Any type can be thrown - integers, strings, objects - but in practice you should throw objects derived from `std::exception`.

```cpp
// throwing a standard exception
throw std::runtime_error("something went wrong");

// technically legal but discouraged
throw 42;
throw "raw string error";
```

A `throw` expression copies (or moves) the thrown object into an implementation-managed storage area, so the original object's lifetime doesn't matter.

## Catching Exceptions

Catch blocks are tried in order, and the first matching handler wins. This is why you always catch **more derived** types before base types.

```cpp
try {
    some_function();
}
catch (const std::invalid_argument& e) {
    // catches invalid_argument specifically
}
catch (const std::exception& e) {
    // catches anything else derived from std::exception
}
catch (...) {
    // catches absolutely anything (int, string, etc.)
}
```

## By Value vs. By Reference

There are three ways to catch:

**By value** (`catch (std::runtime_error e)`) - this **slices** the exception. If the actual thrown object is a derived type, you lose the derived part. Also makes an extra copy. Almost never what you want.

**By reference** (`catch (const std::runtime_error& e)`) - no slicing, no extra copy, polymorphism works correctly. This is the standard practice.

**By pointer** (`catch (const std::exception* e)`) - requires the thrower to `throw new ...`, which creates ownership headaches (who deletes it?). Avoid this.

The guideline is simple: **throw by value, catch by const reference**.

## Re-throwing

Inside a catch block, you can re-throw the current exception with bare `throw;`. This preserves the original dynamic type. If you instead write `throw e;`, you copy/slice.

```cpp
catch (const std::exception& e) {
    log(e.what());
    throw;    // correct: re-throws the original exception
    // throw e; // wrong: slices and copies
}
```

## The Standard Exception Hierarchy

The standard library provides a hierarchy rooted at `std::exception`:

- `std::exception` - base, provides `virtual const char* what() const noexcept`
  - `std::logic_error` - programmer mistakes (violations of preconditions)
    - `std::invalid_argument`, `std::out_of_range`, `std::domain_error`, etc.
  - `std::runtime_error` - errors detectable only at runtime
    - `std::overflow_error`, `std::underflow_error`, `std::range_error`, etc.
  - `std::bad_alloc` - thrown by `new` on allocation failure
  - `std::bad_cast` - thrown by failed `dynamic_cast` on references

## Writing Your Own Exceptions

The cleanest approach is to derive from one of the standard exception classes. You get a working `what()` for free and your exceptions integrate naturally with any code that catches `std::exception&`.

```cpp
class ConnectionError : public std::runtime_error {
public:
    // inherit the constructor that takes a string
    explicit ConnectionError(const std::string& message)
        : std::runtime_error(message)
    {
    }
};

// usage
throw ConnectionError("timeout after 30s");
```

If you need to carry extra data:

```cpp
class ParseError : public std::runtime_error {
public:
    ParseError(const std::string& message, std::size_t line, std::size_t column)
        : std::runtime_error(message)
        , m_line(line)
        , m_column(column)
    {
    }

    [[nodiscard]] std::size_t line() const noexcept {
        return m_line;
    }

    [[nodiscard]] std::size_t column() const noexcept {
        return m_column;
    }

private:
    std::size_t m_line;
    std::size_t m_column;
};
```

If you really need a fully custom hierarchy (rare), you override `what()` yourself:

```cpp
class AppError : public std::exception {
public:
    explicit AppError(std::string message) noexcept
        : m_message(std::move(message)) {}

    [[nodiscard]] const char* what() const noexcept override {
        return m_message.c_str();
    }

private:
    std::string m_message;
};
```

## `noexcept` and Exception Safety

Functions marked `noexcept` promise not to throw. If they do, `std::terminate` is called - no unwinding. This matters for move constructors especially, since containers like `std::vector` will only use your move constructor during reallocation if it's `noexcept`; otherwise they fall back to copying.

```cpp
class Buffer {
public:
    Buffer(Buffer&& other) noexcept;  // enables efficient container operations
};
```

## A Few Important Points

**Stack unwinding**: when an exception propagates, destructors for all local objects in each stack frame are called in reverse order. This is why RAII works - resources get cleaned up even on the error path.

**Exception in destructors**: if a destructor throws while the stack is already unwinding from another exception, `std::terminate` is called. Destructors should be `noexcept` (they are implicitly `noexcept` since C++11 unless you opt out).

**`std::exception_ptr`**: lets you capture an exception and transport it across threads, useful in concurrent code:

```cpp
std::exception_ptr ptr = std::current_exception();
// later, possibly in another thread:
std::rethrow_exception(ptr);
```