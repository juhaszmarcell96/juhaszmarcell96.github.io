# Internal Data Structures of STL Containers

**`std::vector`** - a contiguous dynamic array. Internally it's just three pointers: begin, end (of used elements), and end-of-capacity. When it runs out of capacity it allocates a new block (typically 2× the old size), copies/moves everything over, and frees the old block. This means `push_back` is amortized O(1) but can cause a single expensive reallocation. Iterators and pointers are invalidated on reallocation. Because the memory is contiguous, vector is extremely cache-friendly - this is why it's the default container you should reach for.

**`std::array`** - a fixed-size array allocated on the stack (or wherever the enclosing object lives). No heap allocation, no overhead. Size is a compile-time template parameter. Contiguous, cache-friendly. Essentially a thin wrapper over a C array with `.size()`, iterators, and bounds-checked `.at()`.

**`std::deque`** - not contiguous. Internally it's typically an array of pointers to fixed-size chunks (blocks). This gives O(1) push/pop at both front and back without invalidating elements (though iterators can be invalidated). The chunked layout means it's less cache-friendly than vector but doesn't need to copy everything on growth. Random access is O(1) but with a slightly higher constant than vector because of the indirection.

**`std::list`** - a doubly-linked list. Each node is a separate heap allocation containing the element plus a prev and next pointer. Insertion/removal anywhere is O(1) given an iterator, but traversal is cache-hostile because nodes are scattered in memory. Almost never the right choice in practice - vector with erase-remove is usually faster even for "list-like" workloads because of cache effects.

**`std::forward_list`** - singly-linked list. Same story as list but one pointer per node instead of two, and you can only traverse forward.

**`std::map` / `std::set`** - red-black trees (a self-balancing BST). Every element is a separate node with left/right/parent pointers plus a color bit. Operations are O(log n). Iteration gives sorted order. Like list, each node is a separate allocation, so cache performance is poor. Use when you need sorted order or logarithmic-time lower_bound/upper_bound.

**`std::unordered_map` / `std::unordered_set`** - hash tables. The standard requires a specific structure: an array of buckets, where each bucket is a linked list (separate chaining). This means on collision you get pointer chasing, which is cache-unfriendly. The load factor triggers rehashing (default max load factor is 1.0). Average O(1) lookup, worst case O(n). In practice, many performance-sensitive codebases use open-addressing hash maps (like `absl::flat_hash_map`) because the standard's node-based design has overhead.

---

## Quick Reference Table: Container Complexity

| Container | Access | Search | Insert (end) | Insert (mid) | Memory |
|---|---|---|---|---|---|
| vector | O(1) | O(n) | amortized O(1) | O(n) | contiguous |
| deque | O(1) | O(n) | amortized O(1) | O(n) | chunked |
| list | O(n) | O(n) | O(1) | O(1) at iterator | scattered nodes |
| map/set | O(log n) | O(log n) | O(log n) | O(log n) | tree nodes |
| unordered_map/set | avg O(1) | avg O(1) | avg O(1) | avg O(1) | hash + linked buckets |

---

This covers the core technical material they're likely to probe. Want me to go deeper on any specific topic, or should we move to RAII / templates / exception safety / idioms next?