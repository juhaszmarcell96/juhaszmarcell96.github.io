# Min Stack

```cpp
// =============================================================================
// Min Stack
//
// Key insight: maintain a parallel stack that tracks the running minimum.
// Every push/pop keeps the two stacks in sync so getMin is always O(1).
// =============================================================================

class MinStack {
public:
    void push(int value)
    {
        m_data.push(value);

        // Push onto min stack if it is empty or the new value is <= current min.
        // The <= (not <) is critical: if we push the same minimum twice, we need
        // to track both so that popping one still leaves the correct min.
        if (m_mins.empty() || value <= m_mins.top()) {
            m_mins.push(value);
        }
    }

    void pop()
    {
        if (m_data.empty()) {
            throw std::runtime_error("pop on empty stack");
        }

        // If the value being removed is the current min, pop the min stack too.
        if (m_data.top() == m_mins.top()) {
            m_mins.pop();
        }
        m_data.pop();
    }

    [[nodiscard]] int top() const
    {
        if (m_data.empty()) {
            throw std::runtime_error("top on empty stack");
        }
        return m_data.top();
    }

    [[nodiscard]] int get_min() const
    {
        if (m_mins.empty()) {
            throw std::runtime_error("get_min on empty stack");
        }
        return m_mins.top();
    }

    [[nodiscard]] bool empty() const noexcept
    {
        return m_data.empty();
    }

private:
    std::stack<int> m_data;
    std::stack<int> m_mins;  // parallel stack tracking running minimum
};


void test_min_stack()
{
    std::cout << "--- Min Stack ---\n";

    MinStack s;
    s.push(5);
    assert(s.get_min() == 5);

    s.push(3);
    assert(s.get_min() == 3);

    s.push(3);  // duplicate minimum
    assert(s.get_min() == 3);

    s.push(7);
    assert(s.get_min() == 3);

    s.pop();  // removes 7
    assert(s.get_min() == 3);

    s.pop();  // removes 3 (one of the two)
    assert(s.get_min() == 3);  // second 3 still there

    s.pop();  // removes 3
    assert(s.get_min() == 5);  // back to 5

    std::cout << "All Min Stack tests passed.\n\n";
}
```