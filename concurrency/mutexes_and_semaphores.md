# Mutexes, Semaphores, Lock Guards & Condition Variables in C++

## 1. `std::mutex` - Basic Mutual Exclusion

A mutex protects shared data so only one thread can access it at a time. On its own, you manually lock and unlock it, but this is error-prone (you can forget to unlock, or an exception can skip the unlock).

```cpp
#include <mutex>
#include <iostream>
#include <thread>

std::mutex g_mutex;
int g_counter = 0;

void increment(int times) {
    for (int i = 0; i < times; ++i) {
        g_mutex.lock();
        ++g_counter;
        g_mutex.unlock();
    }
}
```

You almost never want to use raw `.lock()` / `.unlock()` - if the code between them throws, you deadlock. That's where lock guards come in.

---

## 2. `std::lock_guard` - RAII Scoped Locking

This is the workhorse. It locks the mutex on construction, unlocks on destruction. No way to forget, no way to leak a lock through an exception.

```cpp
void increment_safe(int times) {
    for (int i = 0; i < times; ++i) {
        // mutex is locked for the lifetime of this scope
        std::lock_guard<std::mutex> lock(g_mutex);
        ++g_counter;
    }
    // mutex is already released here
}
```

Use `std::lock_guard` when you need to hold the lock for an entire scope and never need to release it early or re-lock.

---

## 3. `std::unique_lock` - Flexible Scoped Locking

Like `lock_guard` but lets you unlock/relock manually, defer locking, and - critically - it's the only lock type that works with condition variables.

```cpp
void flexible_example() {
    std::unique_lock<std::mutex> lock(g_mutex);
    // ... do critical work ...

    lock.unlock();
    // ... do non-critical work without holding the lock ...

    lock.lock();
    // ... back in the critical section ...
}
```

You can also construct it without immediately locking:

```cpp
void deferred_example() {
    std::unique_lock<std::mutex> lock(g_mutex, std::defer_lock);
    // mutex is NOT locked yet

    // ... do some setup ...

    lock.lock();
    // now it is locked, and will auto-unlock on scope exit
}
```

---

## 4. `std::scoped_lock` (C++17) - Locking Multiple Mutexes

When you need to lock more than one mutex at once without risking deadlock (the classic "lock ordering" problem), `std::scoped_lock` handles it. It uses a deadlock-avoidance algorithm internally.

```cpp
std::mutex m_mutex_a;
std::mutex m_mutex_b;

void transfer(int& account_a, int& account_b, int amount) {
    // locks both mutexes atomically, avoids deadlock regardless of call order
    std::scoped_lock lock(m_mutex_a, m_mutex_b);
    account_a -= amount;
    account_b += amount;
}
```

Without this, two threads calling `transfer(a, b, 10)` and `transfer(b, a, 5)` could deadlock if each grabs one mutex first.

---

## 5. `std::condition_variable` - Waiting for a Condition

A condition variable lets a thread sleep until another thread signals that something has changed. It always works together with a `std::mutex` and a `std::unique_lock`.

The classic pattern is a producer-consumer queue:

```cpp
#include <condition_variable>
#include <mutex>
#include <queue>
#include <thread>
#include <iostream>

class ThreadSafeQueue {
private:
    std::queue<int> m_queue;
    std::mutex m_mutex;
    std::condition_variable m_condition;
public:
    void push(int value) {
        {
            std::lock_guard<std::mutex> lock(m_mutex);
            m_queue.push(value);
        }
        // notify AFTER releasing the lock so the woken thread
        // does not immediately block on the mutex
        m_condition.notify_one();
    }

    int pop() {
        std::unique_lock<std::mutex> lock(m_mutex);

        // wait() atomically releases the lock and sleeps.
        // when notified, it re-acquires the lock and checks the predicate.
        // the predicate guards against spurious wakeups.
        m_condition.wait(lock, [this]() { return !m_queue.empty(); });

        int value = m_queue.front();
        m_queue.pop();
        return value;
    }
};
```

Key points:

- **Always use a predicate** with `wait()`. Without it, spurious wakeups (the OS can wake your thread for no reason) will cause bugs.
- **`wait()` requires `std::unique_lock`**, not `lock_guard`, because it needs to unlock/relock the mutex internally.
- **`notify_one()`** wakes one waiting thread, **`notify_all()`** wakes all of them. Use `notify_all` when multiple threads might be able to proceed (e.g. a "shutdown" signal).

---

## 6. Semaphores (C++20) - Counting Access

A semaphore lets up to N threads access a resource concurrently. A binary semaphore (max count of 1) is similar to a mutex, but semaphores are signaling mechanisms - any thread can release, not just the one that acquired.

```cpp
#include <semaphore>
#include <thread>
#include <iostream>
#include <vector>

// allow up to 3 threads in the critical section at once
std::counting_semaphore<3> g_semaphore(3);

void limited_access_worker(int id) {
    g_semaphore.acquire(); // decrements count, blocks if count is 0
    std::cout << "Thread " << id << " entered" << std::endl;

    // ... simulate work ...

    std::cout << "Thread " << id << " leaving" << std::endl;
    g_semaphore.release(); // increments count, potentially unblocks a waiter
}
```

`std::binary_semaphore` is a typedef for `std::counting_semaphore<1>`. A common use is cross-thread signaling where one thread produces and another consumes, but unlike a mutex the "releaser" is not the same thread as the "acquirer":

```cpp
std::binary_semaphore g_signal(0); // starts "unavailable"

void producer() {
    // ... prepare data ...
    g_signal.release(); // signal that data is ready
}

void consumer() {
    g_signal.acquire(); // blocks until producer signals
    // ... use data ...
}
```

---

## When to Use What - Quick Reference

| Scenario | Primitive |
|---|---|
| Protect a shared variable in one scope | `std::lock_guard` |
| Need to unlock/relock, or use with condition variable | `std::unique_lock` |
| Lock 2+ mutexes without deadlock risk | `std::scoped_lock` |
| "Wait until X is true" (producer/consumer, shutdown) | `std::condition_variable` + `unique_lock` |
| Limit concurrent access to N slots | `std::counting_semaphore` |
| Cross-thread one-shot signal (not the same thread locking/unlocking) | `std::binary_semaphore` |

---

## Common Pitfalls

**Data race**: two threads read/write the same variable without synchronization. Even something like `++counter` is not atomic - it's a read-modify-write sequence.

**Deadlock**: Thread A holds mutex 1 and waits for mutex 2, thread B holds mutex 2 and waits for mutex 1. Fix with `std::scoped_lock` or consistent lock ordering.

**Forgetting the predicate on `wait()`**: leads to bugs from spurious wakeups. Always pass a lambda that checks the actual condition.

**Holding a lock while notifying**: technically correct but wasteful - the woken thread immediately contends on the lock. Prefer notifying after releasing the lock (as in the queue example above).

---

## 7. `std::recursive_mutex` - Same Thread Can Lock Multiple Times

A normal `std::mutex` will deadlock if the same thread tries to lock it a second time. A `std::recursive_mutex` keeps an internal ownership count - the owning thread can lock it N times, and it only truly unlocks after N corresponding unlocks.

```cpp
#include <mutex>
#include <iostream>

class TreeNode {
private:
    std::recursive_mutex m_mutex;
    std::vector<TreeNode*> m_children;
    int m_value = 0;
public:
    void process() {
        std::lock_guard<std::recursive_mutex> lock(m_mutex);
        // ... do work on this node ...

        for (auto* child : m_children) {
            // if child->process() somehow calls back into this->process(),
            // a normal mutex would deadlock. recursive_mutex allows it.
            child->process();
        }
    }

    void update_and_process(int value) {
        std::lock_guard<std::recursive_mutex> lock(m_mutex);
        m_value = value;
        // calls process(), which also locks m_mutex -- fine because same thread
        process();
    }
};
```

**When it's useful**: legacy code or complex class hierarchies where public methods call each other and each one needs to be independently thread-safe.

**The catch**: most experienced C++ developers consider `recursive_mutex` a code smell. If you need it, it usually means your locking granularity is wrong. A cleaner pattern is to split into a public method that locks and a private method that does the work unlocked:

```cpp
class CleanerApproach {
private:
    // assumes caller already holds m_mutex
    void process_impl()
    {
        // ... actual work ...
    }

    std::mutex m_mutex;
    int m_value = 0;
public:
    void process() {
        std::lock_guard<std::mutex> lock(m_mutex);
        process_impl();
    }

    void update_and_process(int value) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_value = value;
        process_impl(); // no second lock needed
    }
};
```

---

## 8. `std::shared_mutex` (C++17) - Reader-Writer Lock

Allows multiple concurrent readers OR one exclusive writer. This is a big performance win when reads vastly outnumber writes - which is common in caches, configuration stores, and lookup tables.

```cpp
#include <shared_mutex>
#include <map>
#include <string>

class ConfigStore {
private:
    mutable std::shared_mutex m_mutex;
    std::map<std::string, std::string> m_data;
public:
    std::string get(const std::string& key) const {
        // shared (read) lock -- many threads can hold this simultaneously
        std::shared_lock<std::shared_mutex> lock(m_mutex);
        auto it = m_data.find(key);
        if (it != m_data.end()) {
            return it->second;
        }
        return "";
    }

    void set(const std::string& key, const std::string& value) {
        // unique (write) lock -- exclusive, blocks all readers and writers
        std::unique_lock<std::shared_mutex> lock(m_mutex);
        m_data[key] = value;
    }
};
```

The lock types pair naturally:

| Lock type | Access | Concurrent with other shared locks? | Concurrent with unique lock? |
|---|---|---|---|
| `std::shared_lock` | read | yes | no |
| `std::unique_lock` | write | no | no |

**Pitfall**: `std::shared_mutex` has higher overhead than a plain `std::mutex`. If your critical section is tiny (e.g., reading a single integer), the overhead of the reader-writer machinery can actually make things slower than just using a regular mutex. It pays off when the read-side work is non-trivial or contention is high.

---

## 9. `std::timed_mutex` and `std::recursive_timed_mutex` - Try-Lock with Timeout

These let you attempt to acquire the lock with a deadline, rather than blocking forever. Useful when you want to detect potential deadlocks, implement fallback behavior, or enforce responsiveness guarantees.

```cpp
#include <mutex>
#include <chrono>
#include <iostream>

std::timed_mutex g_mutex;

void try_with_timeout() {
    // try to acquire for at most 100ms
    if (g_mutex.try_lock_for(std::chrono::milliseconds(100))) {
        std::cout << "acquired the lock\n";
        // ... do work ...
        g_mutex.unlock();
    }
    else {
        std::cout << "could not acquire lock in time, doing fallback\n";
    }
}
```

You can also use an absolute time point:

```cpp
void try_with_deadline() {
    auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(2);

    // unique_lock integrates with timed_mutex nicely
    std::unique_lock<std::timed_mutex> lock(g_mutex, deadline);
    if (lock.owns_lock()) {
        // got it
    }
    else {
        // timed out
    }
}
```

`std::recursive_timed_mutex` combines both: same-thread re-entrancy plus timeout support.

---

## 10. `std::shared_timed_mutex` (C++14) - Reader-Writer Lock with Timeouts

Combines the shared/exclusive semantics of `std::shared_mutex` with the timeout capability of `std::timed_mutex`. In practice `std::shared_mutex` (C++17) is preferred when you don't need timeouts, because it can be implemented more efficiently.

```cpp
#include <shared_mutex>
#include <chrono>

std::shared_timed_mutex g_rw_mutex;

bool try_read_with_timeout() {
    if (g_rw_mutex.try_lock_shared_for(std::chrono::milliseconds(50))) {
        // ... read data ...
        g_rw_mutex.unlock_shared();
        return true;
    }
    return false;
}
```

---

## Full Family at a Glance

| Mutex type | Re-entrant? | Shared (reader) mode? | Timed try-lock? |
|---|---|---|---|
| `std::mutex` | no | no | no |
| `std::recursive_mutex` | yes | no | no |
| `std::timed_mutex` | no | no | yes |
| `std::recursive_timed_mutex` | yes | no | yes |
| `std::shared_timed_mutex` (C++14) | no | yes | yes |
| `std::shared_mutex` (C++17) | no | yes | no |

And the lock wrappers that go with them:

| Wrapper | Works with | Key trait |
|---|---|---|
| `std::lock_guard<M>` | any mutex | simplest RAII, one scope |
| `std::unique_lock<M>` | any mutex | flexible, movable, works with condition variables and timed mutexes |
| `std::shared_lock<M>` | shared mutexes | RAII for read-side locking |
| `std::scoped_lock<M...>` | multiple mutexes | deadlock-free multi-lock |

For the interview, the most important ones to know well are `std::mutex`, `std::shared_mutex`, and the reasoning around when `recursive_mutex` is appropriate (and why it's usually a design signal to refactor instead). The timed variants are good to mention if they ask about deadlock detection or timeout strategies.