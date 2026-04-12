# Condition Variables

## The high-level API pattern

```cpp
// Waiting side:
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [&]{ return data_ready; });
// ... use the data ...

// Signaling side:
{
    std::lock_guard<std::mutex> lock(mtx);
    data_ready = true;
}
cv.notify_one();  // or notify_all()
```

## What `cv.wait()` actually does under the hood

The `wait` call does three things in a careful sequence:

1. **Registers itself** as a waiter on the condition variable's internal wait queue.
2. **Releases the mutex** - this is critical, because if you held the mutex while sleeping, nobody could ever change the condition and wake you.
3. **Sleeps** - on Linux, this is a `futex(FUTEX_WAIT, ...)` call on an internal futex word.

When it wakes up (because someone called `notify_one`/`notify_all`):

4. **Reacquires the mutex** - this might itself block if someone else holds it.
5. **Checks the predicate** again (the lambda you passed). If the condition isn't actually true, it goes back to sleep. This is why `wait` takes a predicate - it loops internally.

## Why the predicate loop is necessary

**Spurious wakeups:** the POSIX specification and the C++ standard both allow `wait` to return even if nobody called `notify`. This happens because of implementation details - signals can interrupt the futex call, or the kernel might wake threads for internal scheduling reasons. So you must always recheck.

**Stolen wakeups:** even if the notify was real, between being woken and reacquiring the mutex, another thread might have swooped in, acquired the mutex, and consumed the data. By the time you get the mutex, the condition is false again. The loop handles this too.

## The internal implementation (glibc/pthreads on Linux)

Internally, a `pthread_cond_t` maintains a sequence counter (a futex word). Simplified:

**`pthread_cond_wait`:**
1. Read the current sequence number `seq`.
2. Unlock the associated mutex.
3. Call `futex(FUTEX_WAIT, &seq_addr, seq)` - sleep until the sequence number changes.
4. On wake, relock the mutex.

**`pthread_cond_signal`:**
1. Atomically increment the sequence number.
2. Call `futex(FUTEX_WAKE, &seq_addr, 1)` - wake one waiter.

**`pthread_cond_broadcast`:**
1. Atomically increment the sequence number.
2. Call `futex(FUTEX_WAKE, &seq_addr, INT_MAX)` - wake all waiters.

The sequence number is the trick - it prevents the "lost wake-up" problem where a signal arrives between checking the condition and going to sleep. The futex checks that the value hasn't changed atomically with sleeping.

## Why you need the mutex

Without the mutex, there's a race:

```
Thread A:                          Thread B:
  check condition → false
                                     set condition = true
                                     notify()  ← nobody sleeping yet, lost!
  sleep()  ← sleeps forever
```

The mutex closes this gap. Thread B must hold the mutex to modify the condition, and Thread A holds the mutex while checking the condition and registering itself as a waiter. The mutex creates a happens-before relationship that prevents the lost wake-up.