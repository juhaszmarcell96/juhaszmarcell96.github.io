# LRU Cache

Key insight: std::list supports O(1) splice, and we store iterators into the list inside the hash map. This gives us O(1) get and O(1) put.
- The front of the list = most recently used.
- The back of the list  = least recently used (eviction candidate).

```cpp
class LruCache {
public:
    explicit LruCache(std::size_t capacity)
        : m_capacity{capacity}
    {
    }

    // Returns the value, or -1 if not found.
    // Moves the accessed entry to the front (most recently used).
    [[nodiscard]] int get(int key)
    {
        auto it = m_map.find(key);
        if (it == m_map.end()) {
            return -1;
        }

        // Splice the found node to the front of the list in O(1).
        // This does not invalidate iterators -- the iterator stored
        // in the map remains valid.
        m_order.splice(m_order.begin(), m_order, it->second);
        return it->second->second;
    }

    void put(int key, int value)
    {
        auto it = m_map.find(key);

        if (it != m_map.end()) {
            // Key already exists -- update value and move to front.
            it->second->second = value;
            m_order.splice(m_order.begin(), m_order, it->second);
            return;
        }

        // Evict the least recently used entry if at capacity.
        if (m_map.size() == m_capacity) {
            // The back of the list is the LRU entry.
            int evict_key = m_order.back().first;
            m_order.pop_back();
            m_map.erase(evict_key);
        }

        // Insert new entry at the front.
        m_order.emplace_front(key, value);
        m_map[key] = m_order.begin();
    }

private:
    std::size_t m_capacity;

    // Doubly-linked list of (key, value) pairs.
    // Front = most recently used, back = least recently used.
    std::list<std::pair<int, int>> m_order;

    // Maps key -> iterator into m_order for O(1) lookup + splice.
    std::unordered_map<int, std::list<std::pair<int, int>>::iterator> m_map;
};

void test_lru_cache()
{
    std::cout << "--- LRU Cache ---\n";

    LruCache cache{2};

    cache.put(1, 10);
    cache.put(2, 20);
    assert(cache.get(1) == 10);  // returns 10, moves key 1 to front

    cache.put(3, 30);            // evicts key 2 (LRU)
    assert(cache.get(2) == -1);  // key 2 was evicted
    assert(cache.get(3) == 30);

    cache.put(4, 40);            // evicts key 1
    assert(cache.get(1) == -1);
    assert(cache.get(3) == 30);
    assert(cache.get(4) == 40);

    std::cout << "All LRU Cache tests passed.\n\n";
}
```