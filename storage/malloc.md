# The C Family: `malloc`, `calloc`, `realloc`, `free`

All from `<cstdlib>`. They deal in raw bytes — no constructors, no destructors.

**`malloc(size)`** — allocates `size` bytes, uninitialized (contains garbage). Returns `void*`, or `NULL` on failure.

```cpp
int* p = static_cast<int*>(std::malloc(10 * sizeof(int)));
// p[0..9] exist but contain garbage
std::free(p);
```

**`calloc(count, size)`** — allocates `count * size` bytes, **zero-initialized**. Two advantages over `malloc` + `memset`: it checks for overflow in the multiplication, and the kernel can provide pre-zeroed pages lazily (via copy-on-write mapping of the zero page), so `calloc` for large allocations can be faster than `malloc` + `memset` because pages that are never touched are never physically allocated.

```cpp
int* p = static_cast<int*>(std::calloc(10, sizeof(int)));
// p[0..9] are all 0
std::free(p);
```

**`realloc(ptr, new_size)`** — resizes an existing allocation. This is the interesting one. It tries to extend the block in-place. If it can't (because the memory after the block is in use), it allocates a new block, copies the old data, and frees the old block. Returns the (possibly new) pointer. The old pointer is invalid if realloc moved the data.

```cpp
int* p = static_cast<int*>(std::malloc(10 * sizeof(int)));
p[0] = 42;

// Grow to 20 ints. Might move, might not.
int* q = static_cast<int*>(std::realloc(p, 20 * sizeof(int)));
if (!q)
{
    // realloc failed -- p is still valid!
    std::free(p);
    return 1;
}
p = q;
// p[0] is still 42. p[10..19] are garbage.
std::free(p);
```

Key subtlety: never write `p = realloc(p, new_size)` — if realloc fails and returns `NULL`, you've lost the original pointer and leaked memory.

**`realloc` special cases:**
- `realloc(NULL, size)` behaves like `malloc(size)`
- `realloc(ptr, 0)` is implementation-defined (may free, may not — don't rely on it)

## The C++ Equivalents

C++ separates allocation from construction, which is the key philosophical difference.

| C | C++ equivalent | Notes |
|---|---|---|
| `malloc` | `::operator new(size)` | Raw bytes, throws `std::bad_alloc` on failure |
| `malloc` (nothrow) | `::operator new(size, std::nothrow)` | Returns `nullptr` on failure |
| `calloc` | No direct equivalent | You'd use `operator new` + `std::memset`, or just value-initialize |
| `realloc` | **No equivalent** | This is the big one — see below |
| `free` | `::operator delete(ptr)` | |

**Why C++ has no `realloc`:** `realloc` does a bitwise copy when it moves data. That's fine for `int` and `char`, but it's catastrophic for C++ objects. If you have a `std::string` that contains a pointer to its internal buffer, a bitwise copy creates two objects pointing to the same buffer — then `realloc` frees the original memory. You now have a dangling pointer inside the "moved" string. Any non-trivial C++ type with pointers, references, or internal state cannot be bitwise-copied safely.

This is exactly why `std::vector::reserve` does `operator new` + move-construct each element + destroy originals + `operator delete` the old buffer — it respects object semantics.

**Could we have a C++ realloc?** In theory, for trivially copyable types (`std::is_trivially_copyable_v<T>`), a bitwise realloc is safe. Some allocator libraries expose this (e.g., `jemalloc` has `xallocx` which tries to extend in-place without copying). But the standard doesn't provide it because the general case is unsafe.

## When You'd Actually Use Each

In modern C++, you almost never call any of these directly:

```cpp
// What you actually use:
auto v = std::vector<int>(10);          // instead of malloc + manual tracking
v.resize(20);                           // instead of realloc
auto v2 = std::vector<int>(10, 0);      // instead of calloc

auto p = std::make_unique<int[]>(10);   // instead of malloc + free
```

The only time you touch `operator new` / placement new / explicit destructors is when you're *building* a container or allocator — like we did with the vector example in the previous conversation. Inside your allocator you might even use `malloc`/`free` as the underlying byte source, since at that layer you're below the C++ object model.

## Quick Interview Summary

If asked "what's the difference between malloc, calloc, realloc":
- `malloc` — raw bytes, uninitialized
- `calloc` — zero-initialized, overflow-safe multiplication, potentially lazy zeroing from the OS
- `realloc` — resize, may move, bitwise copy
- C++ has no `realloc` because bitwise copy breaks non-trivial types
- `std::vector` does the "realloc" dance manually: allocate new, move-construct, destroy old, deallocate old — this is the only safe way for arbitrary C++ objects

Want me to add this to the reference file, or move on to another topic?