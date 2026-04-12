# Futex

## What contention means

Contention is when multiple threads try to acquire the same resource (typically a lock) at the same time. If thread A holds a mutex and thread B tries to lock it, B is "contending" for the lock. Uncontended means only one thread wants the lock at any given moment - this is the fast, common case.

Why it matters: the entire design philosophy of futex is to make the uncontended path essentially free (just an atomic instruction in userspace) and only pay the cost of a kernel call when contention actually happens.

## How a futex-based mutex works step by step

Think of a futex as an integer in userspace that the kernel can observe. A simple mutex using futex might use three states:

```
0 = unlocked
1 = locked, no waiters
2 = locked, there are waiters sleeping in the kernel
```

**Lock path:**
1. Try `compare_exchange(0, 1)` - atomically: "if the value is 0, set it to 1." If this succeeds, you got the lock. No syscall. This is just a single CPU instruction (`CMPXCHG` on x86). In the uncontended case you're done here.
2. If it fails (value was 1 or 2), someone else holds the lock. Set the value to 2 (signaling "there are waiters"), then call `futex(FUTEX_WAIT, addr, 2)` - this tells the kernel: "put me to sleep, but only if the value at this address is still 2." That last check avoids a race where the lock was released between your check and your syscall.
3. When you wake up, loop back and try to acquire again.

**Unlock path:**
1. Atomically set the value from whatever it is back to 0.
2. If the old value was 2 (meaning there are waiters), call `futex(FUTEX_WAKE, addr, 1)` to wake one sleeping thread. If the old value was 1 (no waiters), skip the syscall entirely.

So the key insight: the uncontended lock is one atomic instruction, the uncontended unlock is one atomic instruction. No system calls. The kernel is only involved when threads actually need to sleep and wake up, which is the expensive but rare case.

## Why this matters in practice

Before futex existed (pre-2003 Linux), mutexes either always went into the kernel (slow) or busy-spun in userspace (wasted CPU). Futex gave the best of both: userspace fast path, kernel sleep path. This is why `std::mutex` on Linux is fast. In performance-sensitive code like HFT, you might see spin-then-yield strategies where you spin for a few hundred iterations before falling through to the futex sleep, because in very short contention windows spinning can be faster than the syscall.