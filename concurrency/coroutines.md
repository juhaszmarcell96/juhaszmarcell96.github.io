# C++ Coroutines

C++20 coroutines are a language-level mechanism for **suspendable functions**. When a coroutine suspends, it saves its local state and returns control to the caller. When it resumes, it picks up exactly where it left off. No threads are involved unless you explicitly introduce them - a coroutine is purely a control flow abstraction.

---

## How They Differ from Threads

A thread has its own OS-scheduled execution context. Two threads run concurrently (and potentially in parallel), share memory, and need synchronization primitives like mutexes and atomics.

A coroutine is a single function whose execution can be paused and resumed. At any given moment, either the coroutine is running or its caller is running - never both at the same time. There is no data race by construction because there is no concurrent access. Think of it as cooperative multitasking within a single thread: the coroutine explicitly yields control, rather than being preempted.

That said, coroutines *can* be combined with threads. You can resume a coroutine on a different thread, build thread pools that schedule coroutines, or use coroutines as the building blocks of an async runtime. But the coroutine mechanism itself is not concurrent - it is a control flow transformation that the compiler performs.

---

## The Core Language Machinery

A function becomes a coroutine if its body contains any of: `co_await`, `co_yield`, or `co_return`. The compiler then transforms it: it allocates a coroutine frame (holding locals and suspension state), and interacts with a **promise type** that you define to control the coroutine's behavior.

### The three keywords

`co_await expr` - suspend the coroutine. The awaitable `expr` decides whether to actually suspend, what to do while suspended, and what value to produce when resumed.

`co_yield expr` - shorthand for `co_await promise.yield_value(expr)`. Used to produce a value and suspend, as in a generator.

`co_return expr` - finalize the coroutine and deliver a result through the promise.

### The promise type

Every coroutine has an associated promise type that controls its lifecycle. The compiler looks for it via `std::coroutine_traits<ReturnType, Args...>::promise_type`. In practice you define the return type as a class that contains the promise as a nested type:

```cpp
class Generator {
public:
    struct promise_type {
        std::int32_t m_current_value;

        Generator get_return_object() {
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        // suspend immediately on entry -- lazy start
        std::suspend_always initial_suspend() noexcept { return {}; }

        // suspend at the end so the caller can read the final state
        std::suspend_always final_suspend() noexcept { return {}; }

        std::suspend_always yield_value(std::int32_t value) noexcept {
            m_current_value = value;
            return {};
        }

        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };

    // ... handle storage, iteration interface, destructor ...
};
```

`initial_suspend()` - controls whether the coroutine runs eagerly (returns `std::suspend_never`) or lazily (returns `std::suspend_always`) upon creation.

`final_suspend()` - controls what happens when the coroutine finishes. Returning `std::suspend_always` keeps the frame alive so the caller can inspect results or clean up.

`unhandled_exception()` - called if an exception escapes the coroutine body.

### The coroutine handle

`std::coroutine_handle<promise_type>` is a non-owning pointer to the coroutine frame. You use it to resume or destroy the coroutine:

```cpp
m_handle.resume();  // continue execution from the suspension point
m_handle.destroy(); // free the coroutine frame
m_handle.done();    // true if the coroutine has finished
```

You are responsible for calling `destroy()` exactly once. Typically the return type's destructor does this.

### Awaitables

An awaitable is any type that provides three methods:

```cpp
bool await_ready() const noexcept;   // skip suspension if already done?
void await_suspend(std::coroutine_handle<> h); // what to do on suspend
T    await_resume();                 // what value to produce on resume
```

`std::suspend_always` and `std::suspend_never` are trivial awaitables provided by the standard. You write custom ones to integrate with I/O systems, timers, or schedulers.

---

## Common Scenarios

### Generators (lazy sequences)

The most straightforward use case. A function produces values one at a time, suspending between each:

```cpp
Generator fibonacci() {
    std::uint64_t a = 0;
    std::uint64_t b = 1;
    while (true) {
        co_yield a;
        std::uint64_t next = a + b;
        a = b;
        b = next;
    }
}
```

The caller pulls values on demand. No infinite container is allocated - only one value exists at a time. C++23 adds `std::generator<T>` so you do not need to write the boilerplate yourself.

### Async I/O without callback hell

In systems with an event loop or I/O reactor, coroutines replace deeply nested callbacks with straight-line code:

```cpp
// conceptual -- the Task and async functions depend on your framework
Task<std::string> fetch_and_process(Connection& conn) {
    auto request = co_await conn.async_read();
    auto result = co_await process(request);
    co_await conn.async_write(result);
    co_return result;
}
```

The coroutine suspends at each I/O boundary and resumes when the operation completes. The code reads sequentially, but no thread is blocked while waiting. This is how libraries like Asio, libunifex, and various game engines integrate coroutines.

### State machines

A coroutine naturally models a state machine where you suspend at each state transition and resume when the next event arrives. Instead of maintaining explicit state variables and a switch statement, the control flow *is* the state:

```cpp
Task<void> connection_handler(Socket socket) {
    auto handshake = co_await socket.read_handshake();
    if (!validate(handshake)) {
        co_await socket.send_error();
        co_return;
    }

    while (true) {
        auto message = co_await socket.read_message();
        if (message.is_close()) {
            break;
        }
        co_await socket.send_ack();
    }
}
```

### Pipeline / stream processing

You can chain coroutines as producers and consumers, where one coroutine yields values into another that transforms and yields further. This avoids materializing intermediate collections.

---

## What Coroutines Are Not Good For

They do not give you parallelism. If you need CPU-bound work to run on multiple cores, you still need threads (or a thread pool). Coroutines help you *organize* asynchronous control flow, but they do not make things run simultaneously by themselves.

They also come with a learning curve around the promise/awaitable machinery. For simple cases where a plain callback or a future suffices, coroutines may be over-engineering.

---

## Practical Considerations

**Frame allocation** - the coroutine frame is typically heap-allocated, though the compiler can elide this when it can prove the coroutine's lifetime is bounded by the caller (the "Heap Allocation eLision Optimization" - HALO). In latency-sensitive code, you may need custom allocators via `operator new` on the promise type.

**Exception safety** - if an exception propagates out of the coroutine body, `unhandled_exception()` on the promise is called. You decide the policy: store it for later rethrowing, terminate, or log. Think this through for your use case.

**Lifetime pitfalls** - the coroutine frame captures parameters by copy at the point of the coroutine call. If you pass references, the referred-to objects must outlive the coroutine. This is the single most common source of coroutine bugs: a coroutine suspends, the caller's scope ends, and the coroutine resumes with a dangling reference.

**Standard library support** - C++20 provides only the low-level primitives (`coroutine_handle`, `coroutine_traits`, `suspend_always`, `suspend_never`). C++23 adds `std::generator`. For async Task types, you currently rely on libraries or write your own. This is the biggest practical gap - you almost always need a small support library around the raw machinery.