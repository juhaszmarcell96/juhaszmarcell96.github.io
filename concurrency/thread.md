# thread
---

## `std::thread` (C++11)

The basic building block. You create a thread, it starts running immediately, and **you must explicitly join or detach before the `std::thread` object is destroyed** - otherwise the destructor calls `std::terminate()`.

```cpp
#include <thread>
#include <iostream>

void do_work(std::int32_t id) {
    std::cout << "thread " << id << " running" << std::endl;;
}

int main() {
    std::thread t1(do_work, 1);
    std::thread t2(do_work, 2);

    // you MUST do one of these before t1/t2 go out of scope:
    t1.join();   // blocks until t1 finishes
    t2.detach(); // t2 runs independently, no way to join later

    return 0;
}
```

Key points:

- `join()` blocks the calling thread until the target finishes.
- `detach()` releases ownership - the thread runs until it exits on its own.
- Forgetting both is undefined behavior (calls `std::terminate`).
- You can check `t.joinable()` before calling `join()` or `detach()`.

A common RAII guard pattern before `std::jthread` existed:

```cpp
class ThreadGuard {
public:
    explicit ThreadGuard(std::thread& t) : m_thread(t) {}

    ~ThreadGuard() {
        if (m_thread.joinable()) {
            m_thread.join();
        }
    }

    ThreadGuard(const ThreadGuard&) = delete;
    ThreadGuard& operator=(const ThreadGuard&) = delete;

private:
    std::thread& m_thread;
};
```

---

## `std::jthread` (C++20)

"j" stands for "joining." The destructor **automatically requests a stop and then joins**, so you can never accidentally forget. It also has built-in cooperative cancellation via `std::stop_token`.

```cpp
#include <thread>
#include <iostream>
#include <chrono>

void poll_loop(std::stop_token stop_tok, std::int32_t id) {
    while (!stop_tok.stop_requested()) {
        std::cout << "thread " << id << " polling..." << std::endl;;
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
    std::cout << "thread " << id << " stopping gracefully" << std::endl;;
}

int main() {
    {
        std::jthread worker(poll_loop, 42);
        // the stop_token is automatically injected as the first parameter
        // if the callable accepts one

        std::this_thread::sleep_for(std::chrono::seconds(1));

        // optional: you can request stop explicitly
        worker.request_stop();

    } // ~jthread calls request_stop() + join() automatically

    std::cout << "main done" << std::endl;
    return 0;
}
```

Key points:

- No need to manually call `join()` - the destructor handles it.
- `request_stop()` sets a flag; the thread checks it via `stop_token`.
- If your callable's first parameter is `std::stop_token`, `jthread` injects it automatically.
- You can also register callbacks via `std::stop_callback` that fire when stop is requested.

---

## `std::async` / `std::future` (C++11)

Not strictly a "thread variant," but a higher-level abstraction worth knowing. It can spawn a thread (or defer execution) and gives you a `std::future` to retrieve the result.

```cpp
#include <future>
#include <iostream>

std::int64_t compute(std::int32_t x) {
    return static_cast<std::int64_t>(x) * x;
}

int main() {
    // std::launch::async forces a new thread
    // std::launch::deferred would run lazily on .get()
    std::future<std::int64_t> result = std::async(std::launch::async, compute, 42);

    // do other work here...

    // .get() blocks until the result is ready
    std::cout << "result: " << result.get() << std::endl;

    return 0;
}
```

Key points:

- The `std::future` destructor **blocks** if it came from `std::async` and you haven't called `.get()` or `.wait()` yet - this surprises many people.
- No explicit `join()` needed, but the blocking destructor is the implicit equivalent.
- Good for fire-and-forget computations where you want a return value.

---

## POSIX `pthread` comparison

This is the C API that `std::thread` is typically built on top of (on Linux/macOS). It's more verbose, manual, and error-prone - but gives you lower-level control over thread attributes.

```cpp
#include <pthread.h>
#include <cstdio>
#include <cstdint>

struct ThreadArg {
    std::int32_t id;
};

void* thread_func(void* arg) {
    ThreadArg* data = static_cast<ThreadArg*>(arg);
    std::printf("pthread %d running\n", data->id);
    return nullptr;
}

int main() {
    pthread_t t1;
    pthread_t t2;
    ThreadArg arg1{1};
    ThreadArg arg2{2};

    // create threads -- note the void* interface
    pthread_create(&t1, nullptr, thread_func, &arg1);
    pthread_create(&t2, nullptr, thread_func, &arg2);

    // equivalent of std::thread::join
    pthread_join(t1, nullptr);
    pthread_join(t2, nullptr);

    // equivalent of std::thread::detach would be:
    // pthread_detach(t);

    return 0;
}
```

### Side-by-side comparison

| Aspect | `pthread` | `std::thread` | `std::jthread` |
|---|---|---|---|
| Create | `pthread_create(&t, attr, func, arg)` | `std::thread t(func, args...)` | `std::jthread t(func, args...)` |
| Join | `pthread_join(t, &retval)` | `t.join()` | automatic in destructor |
| Detach | `pthread_detach(t)` | `t.detach()` | `t.detach()` (rare) |
| Forget to join | silent resource leak | `std::terminate()` | safe - destructor joins |
| Cancellation | `pthread_cancel(t)` (async, dangerous) | roll your own flag | `stop_token` built in |
| Argument passing | `void*` - manual casting | type-safe variadic | type-safe variadic |
| Return value | `void*` via `pthread_join` | none (use `std::future`) | none (use `std::future`) |
| Thread attributes | `pthread_attr_t` (stack size, policy) | none - use native handle | none - use native handle |
| Mutex | `pthread_mutex_t` + init/destroy | `std::mutex` (RAII via `lock_guard`) | same |

### `pthread` attributes - something C++ doesn't directly expose

```cpp
pthread_attr_t attr;
pthread_attr_init(&attr);

// set stack size to 4 MB
pthread_attr_setstacksize(&attr, 4 * 1024 * 1024);

// set detached at creation time
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

pthread_t t;
pthread_create(&t, &attr, thread_func, &arg);
pthread_attr_destroy(&attr);
```

If you need this from C++, you can access the underlying handle:

```cpp
std::thread t(do_work, 1);
pthread_t native = t.native_handle();
// now you can call pthread_setschedparam, etc.
```

---

### Quick mental model

- **`pthread`** - C API, manual everything, `void*` casting, but full control over attributes and scheduling.
- **`std::thread`** - type-safe C++ wrapper, but you must remember to `join()` or `detach()`.
- **`std::jthread`** - the "safe" version: auto-joins on destruction, built-in stop signaling.
- **`std::async`** - highest level: returns a future, hides the thread, but watch out for the blocking destructor.