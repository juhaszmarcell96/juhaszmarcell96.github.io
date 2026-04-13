# Queue Using Two Stacks

```cpp
// =============================================================================
//
// Key insight: pouring elements from the inbox stack into the outbox stack
// reverses their order, giving us FIFO behavior. We only pour when the
// outbox is empty, so each element is moved at most once -> amortized O(1).
// =============================================================================

class QueueFromStacks {
public:
    void push(int value)
    {
        m_inbox.push(value);
    }

    // Remove and return the front element.
    int pop()
    {
        ensure_outbox();
        if (m_outbox.empty()) {
            throw std::runtime_error("pop on empty queue");
        }
        int result = m_outbox.top();
        m_outbox.pop();
        return result;
    }

    // Peek at the front element without removing it.
    [[nodiscard]] int front()
    {
        ensure_outbox();
        if (m_outbox.empty()) {
            throw std::runtime_error("front on empty queue");
        }
        return m_outbox.top();
    }

    [[nodiscard]] bool empty() const noexcept
    {
        return m_inbox.empty() && m_outbox.empty();
    }

private:
    // Transfer all elements from inbox to outbox, reversing order.
    // Only done when outbox is empty to maintain correct FIFO order.
    void ensure_outbox()
    {
        if (m_outbox.empty()) {
            while (!m_inbox.empty()) {
                m_outbox.push(m_inbox.top());
                m_inbox.pop();
            }
        }
    }

    std::stack<int> m_inbox;   // push side
    std::stack<int> m_outbox;  // pop side
};

void test_queue_from_stacks()
{
    std::cout << "--- Queue From Two Stacks ---\n";

    QueueFromStacks q;
    q.push(1);
    q.push(2);
    q.push(3);

    assert(q.front() == 1);
    assert(q.pop() == 1);
    assert(q.pop() == 2);

    q.push(4);
    q.push(5);

    // Should get 3, 4, 5 in FIFO order.
    assert(q.pop() == 3);
    assert(q.pop() == 4);
    assert(q.pop() == 5);
    assert(q.empty());

    std::cout << "All Queue tests passed.\n\n";
}
```