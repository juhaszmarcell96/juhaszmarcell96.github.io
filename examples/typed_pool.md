# Typed Pool

```cpp
// =============================================================================
// Typed wrapper: allocate/deallocate objects of type T from the pool.
// This shows how you'd use placement new with the raw pool.
// =============================================================================

template <typename T>
class TypedPool {
public:
    explicit TypedPool(std::size_t count)
        : m_pool{sizeof(T), count}
    {
    }

    // Construct a T in pool memory. Returns nullptr if pool is exhausted.
    template <typename... Args>
    [[nodiscard]] T* create(Args&&... args)
    {
        void* mem = m_pool.allocate();
        if (mem == nullptr) {
            return nullptr;
        }
        // Placement new: construct T in the pre-allocated memory.
        return new (mem) T(std::forward<Args>(args)...);
    }

    // Destroy the object and return its memory to the pool.
    void destroy(T* ptr) noexcept
    {
        if (ptr == nullptr) {
            return;
        }
        ptr->~T();  // explicitly call destructor
        m_pool.deallocate(static_cast<void*>(ptr));
    }

private:
    MemoryPool m_pool;
};

// =============================================================================
// Tests
// =============================================================================

void test_typed_pool()
{
    std::cout << "--- Typed Pool ---\n";

    struct Widget {
        int m_id;
        double m_value;

        Widget(int id, double value) : m_id{id}, m_value{value} {}
    };

    TypedPool<Widget> widget_pool{3};

    Widget* w1 = widget_pool.create(1, 3.14);
    Widget* w2 = widget_pool.create(2, 2.71);
    Widget* w3 = widget_pool.create(3, 1.41);

    assert(w1 != nullptr && w1->m_id == 1);
    assert(w2 != nullptr && w2->m_id == 2);
    assert(w3 != nullptr && w3->m_value == 1.41);

    // Pool is full
    assert(widget_pool.create(4, 0.0) == nullptr);

    // Destroy and reuse
    widget_pool.destroy(w2);
    Widget* w4 = widget_pool.create(4, 9.99);
    assert(w4 != nullptr && w4->m_id == 4);

    // Clean up
    widget_pool.destroy(w1);
    widget_pool.destroy(w3);
    widget_pool.destroy(w4);

    std::cout << "All TypedPool tests passed.\n\n";
}
```