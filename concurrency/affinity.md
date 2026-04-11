```cpp
#include <thread>
#include <pthread.h>
#include <sched.h>

[[nodiscard]]
static inline bool pin_thread_to_core (std::thread &t, int core_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);

    int rc = pthread_setaffinity_np(t.native_handle(), sizeof(cpu_set_t), &cpuset);
    return rc == 0;
}

[[nodiscard]]
static inline bool pin_current_thread_to_core(int core_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);

    pthread_t current = pthread_self();
    int rc = pthread_setaffinity_np(current, sizeof(cpu_set_t), &cpuset);
    return rc == 0;
}
```