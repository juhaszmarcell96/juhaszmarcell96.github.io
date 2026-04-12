# POSIX Shared Memory

```cpp
// --- shared_memory_writer.cpp ---
// Compile both files, run writer first, then reader in another terminal.
// g++ -std=c++17 -o shm_writer shared_memory_writer.cpp -lrt -pthread
// g++ -std=c++17 -o shm_reader shared_memory_reader.cpp -lrt -pthread

#include <atomic>
#include <cstdint>
#include <cstring>
#include <fcntl.h>
#include <iostream>
#include <sys/mman.h>
#include <unistd.h>

// Shared data structure -- must be trivially copyable (no pointers, no
// virtual functions, no std::string). Both processes see the same bytes.
struct SharedData {
    std::atomic<std::uint32_t> sequence;    // lock-free synchronization
    std::atomic<bool> writer_done;
    char message[256];
    double values[16];
};

static constexpr const char* shm_name = "/demo_shm";

int main()
{
    // Create shared memory object (appears as /dev/shm/demo_shm on Linux)
    int fd = shm_open(shm_name, O_CREAT | O_RDWR, 0666);
    if (fd == -1)
    {
        std::perror("shm_open");
        return 1;
    }

    // Set size
    if (ftruncate(fd, sizeof(SharedData)) == -1)
    {
        std::perror("ftruncate");
        return 1;
    }

    // Map it
    void* ptr = mmap(
        nullptr, sizeof(SharedData),
        PROT_READ | PROT_WRITE,
        MAP_SHARED,  // both processes see the same physical pages
        fd, 0
    );
    close(fd);

    if (ptr == MAP_FAILED)
    {
        std::perror("mmap");
        return 1;
    }

    // Construct our shared struct in the mapped memory
    // (placement new for proper atomic initialization)
    auto* data = new (ptr) SharedData{};
    data->sequence.store(0, std::memory_order_relaxed);
    data->writer_done.store(false, std::memory_order_relaxed);

    // Write some data
    for (std::uint32_t i = 1; i <= 5; ++i)
    {
        std::snprintf(data->message, sizeof(data->message),
                      "Update #%u from writer (pid %d)", i, getpid());
        for (int j = 0; j < 16; ++j)
        {
            data->values[j] = static_cast<double>(i) * 1.1 + j;
        }
        // Publish: the sequence number tells the reader new data is available.
        // release ensures all writes above are visible before the reader
        // sees the new sequence number.
        data->sequence.store(i, std::memory_order_release);

        std::cout << "Wrote update #" << i << "\n";
        sleep(1);
    }

    data->writer_done.store(true, std::memory_order_release);
    std::cout << "Writer done. Run the reader to see data.\n";

    // Keep alive so reader can see the data
    std::cout << "Press enter to clean up...\n";
    std::cin.get();

    munmap(ptr, sizeof(SharedData));
    shm_unlink(shm_name);  // remove the shared memory object
    return 0;
}
```

```cpp
// --- shared_memory_reader.cpp ---
#include <atomic>
#include <cstdint>
#include <fcntl.h>
#include <iostream>
#include <sys/mman.h>
#include <unistd.h>

struct SharedData {
    std::atomic<std::uint32_t> sequence;
    std::atomic<bool> writer_done;
    char message[256];
    double values[16];
};

static constexpr const char* shm_name = "/demo_shm";

int main()
{
    // Open (not create) the existing shared memory
    int fd = shm_open(shm_name, O_RDONLY, 0);
    if (fd == -1)
    {
        std::perror("shm_open (is the writer running?)");
        return 1;
    }

    void* ptr = mmap(
        nullptr, sizeof(SharedData),
        PROT_READ,     // read-only for the reader
        MAP_SHARED,
        fd, 0
    );
    close(fd);

    if (ptr == MAP_FAILED)
    {
        std::perror("mmap");
        return 1;
    }

    const auto* data = static_cast<const SharedData*>(ptr);

    std::uint32_t last_seen = 0;
    while (!data->writer_done.load(std::memory_order_acquire))
    {
        // acquire ensures we see all the data the writer wrote before
        // publishing the sequence number
        std::uint32_t seq = data->sequence.load(std::memory_order_acquire);
        if (seq != last_seen)
        {
            last_seen = seq;
            std::cout << "Read seq=" << seq
                      << " msg=\"" << data->message << "\""
                      << " values[0]=" << data->values[0] << "\n";
        }
        usleep(100'000);  // poll every 100ms
    }

    std::cout << "Writer is done.\n";
    munmap(const_cast<void*>(ptr), sizeof(SharedData));
    return 0;
}
```

## Shared Memory
- `shm_open` + `mmap(MAP_SHARED)` -- fastest IPC (no copying).
- You must synchronize access yourself (atomics, futex, process-shared mutexes).
- Data structures in shared memory must be trivially representable (no pointers to process-local memory, no vtables, no `std::string`).