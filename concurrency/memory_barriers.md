# Memory Barriers (Memory Fences)

From hardware buffers to C++ memory ordering.

---

## The Problem

Modern CPUs and compilers reorder memory operations for performance. This is invisible to single-threaded code, but in multi-threaded programs it means that one thread's writes may appear in a different order to another thread. Memory barriers are the mechanism to control this.

There are two independent levels of reordering, and you need barriers at both:

1. **Compiler reordering** -- the compiler rearranges loads and stores for optimization.
2. **Hardware reordering** -- the CPU reorders memory operations at runtime.

A correct multi-threaded program must deal with both.

---

## 01 -- Compiler Reordering

The compiler is allowed to reorder any memory operations as long as the observable behavior in a single thread is unchanged. The as-if rule gives it enormous freedom:

```cpp
int x = 0;
int y = 0;

void write() {
    x = 1;     // the compiler may reorder these two stores
    y = 2;     // because neither depends on the other (single-thread view)
}
```

A compiler barrier prevents this:

```cpp
// GCC/Clang compiler barrier (not a hardware barrier)
asm volatile("" ::: "memory");

// MSVC compiler barrier
_ReadWriteBarrier();

// Portable C++ way: std::atomic_signal_fence
std::atomic_signal_fence(std::memory_order_seq_cst);
```

The `asm volatile("" ::: "memory")` tells the compiler "assume all memory has been clobbered" -- so it must flush all pending writes to memory and re-read any values it had cached in registers. It emits zero CPU instructions -- it only constrains the compiler's optimizer.

But this is rarely used directly. `std::atomic` operations implicitly include the right compiler barriers.

---

## 02 -- Hardware Reordering (The Interesting Part)

Even if the compiler emits instructions in the right order, the CPU may still reorder them at runtime. To understand why, you need to know how writes and reads actually travel through the hardware.

### The store buffer

When a CPU core executes a store instruction, the write does not go directly to the cache. It goes into a **store buffer** -- a small queue private to that core. This allows the core to keep executing without waiting for the cache hierarchy to process the write.

```
Core 0                      Core 1
  |                            |
  v                            v
[Store Buffer]             [Store Buffer]
  |                            |
  v                            v
[L1 Cache] <-- coherency --> [L1 Cache]
  |                            |
  v                            v
[L2 Cache] <-- coherency --> [L2 Cache]
  |                            |
  +-----------+  +-------------+
              v  v
          [L3 / Memory]
```

The store buffer is invisible to other cores. Core 1 cannot see Core 0's store buffer. A write is only visible to other cores once it drains from the store buffer into the cache, at which point the cache coherency protocol (MESI/MOESI) propagates it.

### The invalidation queue

On the read side, when a core receives an invalidation message (another core wrote to a cache line it holds), it may not process it immediately. Instead, it can queue the invalidation and acknowledge it right away. This means the core might read stale data from its own cache for a brief window.

```
Incoming invalidation    --> [Invalidation Queue] --> Process later
                          (acknowledged immediately)
```

### How reordering happens

These two buffers create four types of reordering:

**Store-Store reordering:** Two writes from the same core might reach the cache (and thus become visible to other cores) in a different order than they were issued, if the store buffer drains out of order. Most architectures (x86 included) do NOT reorder stores with respect to other stores. ARM and POWER do.

**Load-Load reordering:** Two reads might execute out of order because one hits the L1 cache and the other misses. The CPU may complete the cache hit first. Also, a stale value in the invalidation queue can make a later read appear to have happened before an earlier one. ARM and POWER allow this. x86 does not (mostly).

**Store-Load reordering:** A load that comes after a store (in program order) might execute before the store's value has drained from the store buffer to the cache. The load goes straight to cache and gets the old value, while the store is still sitting in the buffer. **This is the one reordering that ALL architectures allow, including x86.** It is the most common source of multi-threaded bugs.

**Load-Store reordering:** A store might be committed before an earlier load completes. Rare, but possible on weaker architectures (ARM, POWER).

### What a hardware memory barrier does

A hardware memory barrier forces the CPU to flush or drain these buffers at a specific point:

**Store fence (sfence on x86):** Drains the store buffer. All stores before the fence become visible to other cores before any store after the fence.

**Load fence (lfence on x86):** Waits for the invalidation queue to be processed. All loads after the fence see the most recent values.

**Full fence (mfence on x86):** Both. Drains the store buffer AND processes pending invalidations. This is the most expensive barrier.

```
Core 0 timeline:

  store x = 1
  store y = 2
  --- STORE FENCE ---     <- forces x=1 and y=2 to drain to cache
  store z = 3             <- z=3 will not be visible before x=1 and y=2

Core 1 timeline:

  --- LOAD FENCE ---      <- forces invalidation queue to be processed
  read y                  <- guaranteed to see the latest value
  read x                  <- guaranteed to see the latest value
```

### x86 is "strong" but not sequential

x86 has a relatively strong memory model called Total Store Order (TSO). It prevents most reorderings:

- Stores are not reordered with other stores (store-store: NO)
- Loads are not reordered with other loads (load-load: NO)
- Loads are not reordered with older stores to the same address (no issue)
- **Stores CAN be reordered with younger loads** (store-load: YES)

This last point means that even on x86, you can observe another core's store as happening "after" your own load, even if the store was issued first. The classic example:

```
Initially: x = 0, y = 0

Core 0:             Core 1:
  x = 1               y = 1
  r0 = y              r1 = x

Can we get r0 == 0 AND r1 == 0?
On x86: YES, because of store-load reordering.
Both loads can execute before the stores drain from their store buffers.
```

### ARM/POWER are "weak"

ARM and POWER allow almost all reorderings. Every non-trivial multi-threaded pattern needs explicit barriers. This is why code that "works on x86" may break spectacularly on ARM.

---

## 03 -- From Hardware to C++: std::memory_order

C++ provides a portable abstraction over these hardware differences through `std::memory_order`. You attach an ordering to each atomic operation, and the compiler emits the right barriers for the target architecture.

### memory_order_relaxed

No ordering guarantees at all. The atomic operation itself is atomic (no torn reads/writes), but there are no barriers. Other threads may see operations in any order.

```cpp
std::atomic<int> counter{0};

// Just increment atomically, don't care about ordering
counter.fetch_add(1, std::memory_order_relaxed);
```

**Hardware cost:** Just the atomic instruction (lock-free increment). No fences emitted. On x86: `lock xadd`. On ARM: `ldxr/stxr` loop without barriers.

**Use case:** Statistics counters, reference counts where you don't need to synchronize other data.

### memory_order_acquire (on loads)

All reads and writes after this load are guaranteed to happen after it. Nothing above can "sink" below the acquire. This is a load fence.

```cpp
std::atomic<bool> ready{false};
int data = 0;

// Producer (different thread):
data = 42;
ready.store(true, std::memory_order_release);

// Consumer:
while (!ready.load(std::memory_order_acquire)) {}
// At this point, data is guaranteed to be 42.
// The acquire prevents reading data before reading ready==true.
int value = data;   // guaranteed to see 42
```

**Hardware cost:** On x86, acquire is essentially free because x86 doesn't reorder loads past loads. The compiler just emits a compiler barrier. On ARM, it emits a `dmb ishld` (data memory barrier) or uses `ldar` (load-acquire instruction).

### memory_order_release (on stores)

All reads and writes before this store are guaranteed to happen before it. Nothing below can "float" above the release. This is a store fence.

```cpp
// Producer:
data = 42;                                      // must complete before the release
ready.store(true, std::memory_order_release);   // release: flushes prior writes
```

**Hardware cost:** On x86, release is essentially free because x86 doesn't reorder stores past stores. Compiler barrier only. On ARM, it emits a `dmb ish` or uses `stlr` (store-release instruction).

### Acquire-Release pairing

Acquire and release form a synchronized pair. A release-store in one thread "synchronizes with" an acquire-load in another thread. Everything before the release is visible to everything after the acquire:

```
Thread A (producer):      |  Thread B (consumer):
                          |     
write data = 42           |     
write metadata = "info"   |
  |                       |
  v                       |
RELEASE store ready=true  |
                          |   ACQUIRE load ready==true
                          |     |
                          |     v
                          |   read data      -> guaranteed 42
                          |   read metadata  -> guaranteed "info"
```

This is the most common pattern. It's how mutexes work internally: `lock()` is an acquire, `unlock()` is a release.

### memory_order_acq_rel

Both acquire and release on a single read-modify-write operation (like `compare_exchange` or `fetch_add`). It acts as an acquire for the read part and a release for the write part.

```cpp
std::atomic<int> flag{0};

// This is both a load (acquire) and a store (release)
int old = flag.fetch_add(1, std::memory_order_acq_rel);
```

### memory_order_seq_cst (the default)

Sequential consistency. The strongest ordering. All `seq_cst` operations across all threads appear to happen in some single total order that is consistent with the program order of each thread.

```cpp
std::atomic<int> x{0};
x.store(1);                               // default is seq_cst
int val = x.load();                       // default is seq_cst
int old = x.fetch_add(1);                 // default is seq_cst
```

**Hardware cost:** On x86, a `seq_cst` store emits an `mfence` (or `xchg`, which has implicit full-barrier semantics). This is the store-load barrier that even x86 needs. On ARM, full barriers (`dmb ish`) around the operation.

**Why it's expensive:** It's the only ordering that prevents store-load reordering on x86. All other orderings are free on x86 at the hardware level. On ARM, every ordering above relaxed has a cost.

### The cost hierarchy

From cheapest to most expensive:

```
relaxed      -- no barrier (just atomic operation)
acquire      -- load barrier (free on x86, cheap-ish on ARM)
release      -- store barrier (free on x86, cheap-ish on ARM)
acq_rel      -- both (free on x86 for most ops, moderate on ARM)
seq_cst      -- full barrier (mfence on x86, full dmb on ARM)
```

---

## 04 -- How the Cache Coherency Protocol Fits In

Memory barriers control *when* writes become visible. The cache coherency protocol (MESI/MOESI) controls *how* they propagate once visible. These are complementary:

1. Core 0 writes `x = 1`. The write sits in the store buffer.
2. A **store barrier** forces the store buffer to drain. The write enters Core 0's L1 cache.
3. The cache coherency protocol detects that Core 1 also holds cache line containing `x`.
4. It sends an **invalidation message** to Core 1.
5. Core 1 receives the invalidation (or processes it from the invalidation queue after a load barrier).
6. Next time Core 1 reads `x`, it cache-misses, fetches the updated line, and sees `x == 1`.

The barrier flushes the local buffer. The coherency protocol propagates the change globally. Without the barrier, the write might sit in the store buffer indefinitely (in practice, it drains quickly, but "quickly" is not "guaranteed before the next instruction").

### MESI states (simplified)

Each cache line in each core is in one of four states:

- **Modified:** This core has the only copy, and it's dirty (different from memory). Other cores must ask this core for the data.
- **Exclusive:** This core has the only copy, and it's clean. Can transition to Modified without bus traffic.
- **Shared:** Multiple cores have clean copies. Must invalidate others before writing.
- **Invalid:** This core's copy is stale. Must fetch from another core or memory.

When Core 0 writes to a Shared line, it sends invalidations to all other cores. Once they acknowledge, Core 0 transitions the line to Modified. This is the mechanism that makes writes visible -- but only after they leave the store buffer and enter the cache.

---

## 05 -- Putting It All Together: A Concrete Example

A simple spinlock to see all the layers:

```cpp
class SpinLock {
private:
    std::atomic<bool> m_locked{false};
public:
    void lock() {
        // Acquire: once we see locked==false, all prior writes
        // by the previous lock holder are visible to us.
        while (m_locked.exchange(true, std::memory_order_acquire)) {
            // Spin. The exchange is atomic (no torn read/write).
            // On x86: lock xchg (implicit full barrier, but we only need acquire).
            // On ARM: ldaxr/stxr loop with acquire semantics.
        }
    }

    void unlock() {
        // Release: all writes we did while holding the lock
        // are guaranteed to be visible before the lock is released.
        m_locked.store(false, std::memory_order_release);
        // On x86: plain mov (release is free on x86 for stores).
        // On ARM: stlr (store-release).
    }
};
```

What happens at the hardware level when Thread B acquires after Thread A releases:

```
Thread A (holding lock):
  writes to shared data...
  m_locked.store(false, release)
    -> compiler barrier: compiler won't move data writes past this store
    -> store buffer: the false value enters the store buffer
    -> release semantics: all prior stores drain before (or with) this one
    -> cache coherency: once in cache, invalidation sent to Thread B's line

Thread B (spinning):
  m_locked.exchange(true, acquire)
    -> sees false (lock acquired)
    -> acquire semantics: processes invalidation queue
    -> all subsequent reads see Thread A's writes
    -> reads shared data: sees Thread A's values
```

---

## 06 -- Common Patterns and Their Orderings

### Flag / ready signal (acquire-release)

```cpp
std::atomic<bool> ready{false};
int payload = 0;

// Producer
payload = 42;
ready.store(true, std::memory_order_release);

// Consumer
while (!ready.load(std::memory_order_acquire)) {}
assert(payload == 42);   // guaranteed
```

### Reference counting (relaxed increment, acq_rel or seq_cst on decrement)

```cpp
std::atomic<int> ref_count{1};

void add_ref() {
    // Relaxed: we don't need to synchronize any other data on increment
    ref_count.fetch_add(1, std::memory_order_relaxed);
}

void release() {
    // The decrement needs acquire-release:
    // - release: all writes to the object are visible before we check if count==0
    // - acquire: if we're the one who drops to zero, we need to see all other
    //   threads' writes before we destroy the object
    if (ref_count.fetch_sub(1, std::memory_order_acq_rel) == 1) {
        // We decremented from 1 to 0. We're the last reference.
        // The acq_rel ensures we see all modifications from all threads.
        delete the_object;
    }
}
```

### Double-checked locking (acquire-release)

```cpp
std::atomic<Widget*> instance { nullptr };
std::mutex mtx;

Widget* get_instance() {
    Widget* ptr = instance.load(std::memory_order_acquire);
    if (ptr == nullptr) {
        std::lock_guard<std::mutex> lock(mtx);
        ptr = instance.load(std::memory_order_relaxed);  // re-check under lock
        if (ptr == nullptr) {
            ptr = new Widget();
            instance.store(ptr, std::memory_order_release);
        }
    }
    return ptr;
}
```

---

## 07 -- Key Takeaways

**Two levels of reordering:** compiler (prevented by compiler barriers) and hardware (prevented by CPU fence instructions). `std::atomic` with memory orderings handles both portably.

**Store buffers are the root cause:** Writes are not instantly visible to other cores. They sit in a private buffer until drained. Barriers force the drain.

**Invalidation queues delay reads:** A core may read stale data from its cache if it hasn't processed incoming invalidations yet. Load barriers force processing.

**Acquire-release is the workhorse pattern.** Most multi-threaded synchronization boils down to: release to publish, acquire to observe. Mutexes, spinlocks, and flag-based signaling all use this.

**seq_cst is the safe default but costs more.** On x86, it's the only ordering that emits an actual hardware barrier (mfence). On ARM, all non-relaxed orderings have a cost. Use acquire-release when you can reason about the specific synchronization relationship.

**x86's strong model is a trap.** Code that works on x86 without proper barriers may break on ARM. Always program against the C++ memory model, not the hardware model.

**Cache coherency is your friend.** Once a write makes it into the cache, the coherency protocol guarantees it will propagate. The barrier's job is just to get the write out of the store buffer and into the cache at the right time.