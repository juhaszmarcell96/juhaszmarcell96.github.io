# Dining Philosophers

Five philosophers sit around a table with a fork between each pair.
To eat, a philosopher needs both adjacent forks. Naive approach
(everyone picks up left fork first) can deadlock.

Solution: resource ordering. Each philosopher picks up the
LOWER-numbered fork first. This breaks the circular wait because
philosopher 4 (between forks 4 and 0) picks up fork 0 first,
not fork 4 -- so the cycle is broken.

**Alternative: semaphore approach** Allow at most 4 philosophers to attempt eating at once.
If only 4 can try, at least one of them will always be able to get
both forks (pigeonhole principle), so no deadlock.

```cpp
// =============================================================================
//
// Deadlock requires all four conditions:
//   1. Mutual exclusion   (forks are exclusive)
//   2. Hold and wait      (hold one fork, wait for another)
//   3. No preemption      (can't steal a fork)
//   4. Circular wait      <-- broken by ordering
//
// Alternative shown: semaphore limiting concurrency to 4 philosophers.
// =============================================================================

class DiningPhilosophers {
private:
    std::array<std::mutex, 5> m_forks;
public:
    // Each philosopher calls this with their id (0-4).
    // The callbacks simulate picking up/putting down forks and eating.
    void dine(
        int philosopher_id,
        std::function<void(int)> pick_up,
        std::function<void(int)> put_down,
        std::function<void(int)> eat)
    {
        int left_fork = philosopher_id;
        int right_fork = (philosopher_id + 1) % 5;

        // Always pick up the lower-numbered fork first.
        int first = std::min(left_fork, right_fork);
        int second = std::max(left_fork, right_fork);

        m_forks[first].lock();
        pick_up(first);

        m_forks[second].lock();
        pick_up(second);

        eat(philosopher_id);

        m_forks[second].unlock();
        put_down(second);

        m_forks[first].unlock();
        put_down(first);
    }
};

// 
// 
// 
// 

class DiningPhilosophersSemaphore {
private:
    std::array<std::mutex, 5> m_forks;
    std::counting_semaphore<4> m_seats{4};  // max 4 concurrent diners
public:
    void dine(
        int philosopher_id,
        std::function<void(int)> pick_up,
        std::function<void(int)> put_down,
        std::function<void(int)> eat)
    {
        // At most 4 philosophers may enter the "trying to eat" section.
        m_seats.acquire();

        int left_fork = philosopher_id;
        int right_fork = (philosopher_id + 1) % 5;

        // No special ordering needed -- the semaphore prevents deadlock.
        m_forks[left_fork].lock();
        pick_up(left_fork);

        m_forks[right_fork].lock();
        pick_up(right_fork);

        eat(philosopher_id);

        m_forks[right_fork].unlock();
        put_down(right_fork);

        m_forks[left_fork].unlock();
        put_down(left_fork);

        m_seats.release();
    }
};

// =============================================================================
// Tests
// =============================================================================

void test_dining_philosophers()
{
    std::cout << "--- Dining Philosophers (resource ordering) ---\n";

    DiningPhilosophers table;
    std::atomic<int> eat_count{0};
    std::mutex log_mutex;

    constexpr int rounds = 100;

    auto philosopher_task = [&](int id) {
        for (int r = 0; r < rounds; ++r) {
            table.dine(
                id,
                [&](int fork) {
                    // pick_up callback (no-op for stress test)
                    (void)fork;
                },
                [&](int fork) {
                    // put_down callback
                    (void)fork;
                },
                [&](int phil_id) {
                    // eat callback
                    (void)phil_id;
                    ++eat_count;
                });
        }
    };

    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back(philosopher_task, i);
    }
    for (auto& t : threads) {
        t.join();
    }

    assert(eat_count == 5 * rounds);
    std::cout << "  All 5 philosophers ate " << rounds
              << " times each (total: " << eat_count << ").\n";
    std::cout << "  No deadlock detected. Passed.\n\n";
}

void test_dining_philosophers_semaphore()
{
    std::cout << "--- Dining Philosophers (semaphore) ---\n";

    DiningPhilosophersSemaphore table;
    std::atomic<int> eat_count{0};

    constexpr int rounds = 100;

    auto philosopher_task = [&](int id) {
        for (int r = 0; r < rounds; ++r) {
            table.dine(
                id,
                [](int) {},
                [](int) {},
                [&](int) { ++eat_count; });
        }
    };

    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back(philosopher_task, i);
    }
    for (auto& t : threads) {
        t.join();
    }

    assert(eat_count == 5 * rounds);
    std::cout << "  Total eats: " << eat_count << ". No deadlock. Passed.\n\n";
}
```