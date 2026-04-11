```cpp
#include <atomic>
#include <thread>

class Spinlock {
private:
    std::atomic_flag m_flag = ATOMIC_FLAG_INIT;
public:
    void lock() noexcept {
        int backoff = 1;
        constexpr int MAX_BACKOFF = 64;
        
        while (m_flag.test_and_set(std::memory_order_acquire)) {
            // exponential backoff
            for (int i = 0; i < backoff; ++i) {
                // use more aggressive hardware-specific instructions for better performance
                std::this_thread::yield();
            }
            backoff = (backoff < MAX_BACKOFF) ? backoff * 2 : MAX_BACKOFF;
        }
    }
    
    bool try_lock() noexcept {
        return !m_flag.test_and_set(std::memory_order_acquire);
    }
    
    void unlock() noexcept {
        m_flag.clear(std::memory_order_release);
    }
};

// RAII guard compatible with std::lock_guard interface
template<typename Lock>
class LockGuard {
private:
    Lock& m_lock;
    bool m_owns;
public:
    explicit LockGuard(Lock& lock) : m_lock(lock), m_owns(true) {
        m_lock.lock();
    }
    
    LockGuard(Lock& lock, std::defer_lock_t) : m_lock(lock), m_owns(false) {}
    
    ~LockGuard() {
        if (m_owns) {
            m_lock.unlock();
        }
    }
    
    bool try_lock() noexcept {
        if (!m_owns && m_lock.try_lock()) {
            m_owns = true;
            return true;
        }
        return false;
    }
    
    LockGuard(const LockGuard&) = delete;
    LockGuard& operator=(const LockGuard&) = delete;
};
```