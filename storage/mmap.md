# `mmap`

## Memory-Mapping a File

```cpp
#include <cerrno>
#include <cstdio>
#include <cstring>
#include <fcntl.h>
#include <iostream>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main() {
    // -- Read a file by mapping it into memory --
    const char* path = "/etc/hostname";
    int fd = open(path, O_RDONLY);
    if (fd == -1) {
        std::perror("open");
        return 1;
    }

    // Get file size
    struct stat st{};
    if (fstat(fd, &st) == -1) {
        std::perror("fstat");
        close(fd);
        return 1;
    }
    const std::size_t file_size = static_cast<std::size_t>(st.st_size);

    // Map the file into our address space
    // MAP_PRIVATE = copy-on-write (our writes don't affect the file)
    void* mapped = mmap(
        nullptr,           // let the kernel choose the address
        file_size,         // length
        PROT_READ,         // read-only access
        MAP_PRIVATE,       // private copy-on-write mapping
        fd,                // file descriptor
        0                  // offset in file
    );
    if (mapped == MAP_FAILED) {
        std::perror("mmap");
        close(fd);
        return 1;
    }

    // We can close the fd now -- the mapping keeps the file open internally
    close(fd);

    // Use it like a regular pointer
    const char* data = static_cast<const char*>(mapped);
    std::cout << "File contents (" << file_size << " bytes):\n";
    std::cout.write(data, static_cast<std::streamsize>(file_size));

    // Hint to the kernel about access pattern
    // MADV_SEQUENTIAL -> aggressive read-ahead
    madvise(mapped, file_size, MADV_SEQUENTIAL);

    // Unmap when done
    munmap(mapped, file_size);

    // -- Create and write a file via mmap --
    const char* out_path = "/tmp/mmap_demo.bin";
    constexpr std::size_t out_size = 4096;

    int out_fd = open(out_path, O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (out_fd == -1) {
        std::perror("open out");
        return 1;
    }

    // Must set the file size first -- mmap doesn't extend files
    if (ftruncate(out_fd, static_cast<off_t>(out_size)) == -1) {
        std::perror("ftruncate");
        close(out_fd);
        return 1;
    }

    void* out_mapped = mmap(
        nullptr, out_size,
        PROT_READ | PROT_WRITE,
        MAP_SHARED,        // MAP_SHARED = writes go to the file
        out_fd, 0
    );
    close(out_fd);

    if (out_mapped == MAP_FAILED) {
        std::perror("mmap out");
        return 1;
    }

    // Write directly into the file via the pointer
    char* out_data = static_cast<char*>(out_mapped);
    std::strcpy(out_data, "Hello from mmap!\n");

    // Force write to disk (otherwise it happens lazily)
    msync(out_mapped, out_size, MS_SYNC);

    munmap(out_mapped, out_size);
    std::cout << "Wrote to " << out_path << "\n";

    return 0;
}
```

---

## `mmap` (Anonymous) vs `sbrk`

```cpp
#include <cstdio>
#include <cstring>
#include <iostream>
#include <sys/mman.h>
#include <unistd.h>

int main() {
    // -- sbrk: the old-school heap --
    // sbrk(0) returns the current "program break" (end of the data segment).
    // sbrk(n) moves it forward by n bytes, returning the old break.
    // This is what traditional malloc uses internally for small allocations.

    void* old_break = sbrk(0);
    std::cout << "Current program break: " << old_break << "\n";

    // Allocate 1024 bytes by extending the data segment
    void* allocated = sbrk(1024);
    if (allocated == reinterpret_cast<void*>(-1)) {
        std::perror("sbrk");
        return 1;
    }

    void* new_break = sbrk(0);
    std::cout << "After sbrk(1024):     " << new_break << "\n";
    std::cout << "Got memory at:        " << allocated << "\n";

    // We can use it
    std::memset(allocated, 0, 1024);

    // "Free" by moving the break back (only works for LIFO-style usage)
    sbrk(-1024);
    std::cout << "After sbrk(-1024):    " << sbrk(0) << "\n";
    // In practice, nobody uses sbrk directly. malloc does.

    // -- Anonymous mmap: the modern alternative --
    // No file backing. The kernel gives you zero-filled pages.
    // This is what malloc uses for large allocations (typically > 128KB).

    constexpr std::size_t size = 1024 * 1024;  // 1 MiB
    void* region = mmap(
        nullptr, size,
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS,  // no file, private to this process
        -1, 0                         // fd=-1, offset=0 for anonymous
    );
    if (region == MAP_FAILED) {
        std::perror("mmap");
        return 1;
    }

    std::cout << "\nmmap gave us 1 MiB at: " << region << "\n";

    // The pages are lazily allocated -- physical RAM isn't used until we
    // touch each page (demand paging). This write triggers a page fault
    // for the first page:
    static_cast<char*>(region)[0] = 'A';

    // Advantages over sbrk:
    //   - Can allocate/free anywhere (not just the end of the heap)
    //   - Can set permissions (PROT_READ only, PROT_NONE for guard pages)
    //   - Can return memory to the OS immediately (munmap)
    //   - Thread-safe (sbrk is not)

    // Return memory to the OS
    munmap(region, size);

    // -- Guard pages: detect buffer overflows --
    constexpr std::size_t page_size = 4096;
    void* guarded = mmap(
        nullptr, page_size * 2,
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS, -1, 0
    );
    // Make the second page inaccessible
    mprotect(
        static_cast<char*>(guarded) + page_size,
        page_size,
        PROT_NONE  // any access -> SIGSEGV
    );
    // Now writing past the first page will segfault immediately
    // instead of silently corrupting memory.
    munmap(guarded, page_size * 2);

    return 0;
}
```

## mmap
- **File-backed**: `mmap(NULL, size, PROT_*, MAP_SHARED|MAP_PRIVATE, fd, offset)`
- **Anonymous**: `MAP_ANONYMOUS | MAP_PRIVATE`, fd=-1. Like malloc but you control the lifetime.
- `MAP_SHARED` vs `MAP_PRIVATE`: shared writes back to file/shared with other mmaps. Private is copy-on-write.
- `mprotect` to change page permissions. `madvise` for access pattern hints. `msync` to flush to disk.

## sbrk
- `sbrk(n)` extends the data segment. Returns old break. Only used inside malloc implementations.
- Single-threaded, LIFO-only freeing, obsolete. Know what it does conceptually.