# `fork()` Basics

```cpp
#include <cstdio>
#include <iostream>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    int x = 42;

    pid_t pid = fork();
    // After fork(), there are now TWO processes running.
    // Both continue from this exact point.
    // The ENTIRE virtual address space is duplicated (copy-on-write).

    if (pid < 0) {
        std::perror("fork");
        return 1;
    }
    else if (pid == 0) {
        // Child process
        // It has its OWN copy of x (copy-on-write -- the physical page
        // is shared until one process writes to it)
        x = 100;
        std::cout << "[child]  pid=" << getpid()  << " x=" << x << "\n";
        _exit(0);  // use _exit in child, not exit() (avoids flushing
                   // parent's stdio buffers -- more on this below!)
    }
    else {
        // Parent process
        // pid is the child's PID
        std::cout << "[parent] pid=" << getpid() << " child=" << pid
                  << " x=" << x << "\n";  // still 42
        waitpid(pid, nullptr, 0);  // wait for child to finish
    }

    return 0;
}
```

---

## The Fork + Printf Buffering Puzzle

This is the classic interview question. There are two interacting effects:

**Effect 1: fork in a loop** -- each fork doubles the number of processes.

**Effect 2: stdio buffering** -- `printf` (and `std::cout` when tied to a
non-terminal) uses a *userspace buffer*. This buffer lives in the process's
address space. When you `fork()`, the child gets a *copy* of that buffer,
including any unflushed content. When both parent and child eventually flush
(on `exit`, newline, or explicit `fflush`), the buffered text is printed *twice*.

### Fork in a loop -- how many processes?

```cpp
#include <cstdio>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    // How many times does "hello" get printed?
    for (int i = 0; i < 3; ++i) {
        fork();
        printf("hello\n");
    }

    // Let's trace it:
    //
    // i=0: 1 process calls fork() -> 2 processes. Both print "hello\n". (2 prints)
    // i=1: 2 processes each call fork() -> 4 processes. All print "hello\n". (4 prints)
    // i=2: 4 processes each call fork() -> 8 processes. All print "hello\n". (8 prints)
    //
    // Total: 2 + 4 + 8 = 14 prints.
    //
    // General formula for N iterations: sum of 2^(i+1) for i=0..N-1 = 2^(N+1) - 2
    // For N=3: 2^4 - 2 = 14.

    // Wait for all children to avoid zombies
    while (wait(nullptr) > 0) {}
    return 0;
}
```

### The buffering trap -- `printf` WITHOUT newline

```cpp
#include <cstdio>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    // KEY DIFFERENCE: no \n in the printf!
    // How many times does "hello" appear in the output?
    printf("hello");   // <-- NO newline. Stays in the stdio buffer.
    fork();

    // After fork(), both parent and child have "hello" sitting in their
    // stdio buffer (because the buffer was copied with the address space).
    //
    // When each process exits, its stdio buffers are flushed.
    // So both parent AND child print "hello".
    //
    // Output: "hellohello"
    //
    // If we had written printf("hello\n"), the \n would have caused an
    // immediate flush (when stdout is a terminal, it's line-buffered).
    // The buffer would be empty at fork time, and only the parent's
    // "hello\n" would appear. Output: "hello"

    // Wait for child
    while (wait(nullptr) > 0) {}
    return 0;
}
```

### The full tricky version -- fork in a loop WITHOUT newline

```cpp
#include <cstdio>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    // The classic interview question:
    for (int i = 0; i < 2; ++i) {
        printf("A");   // NO newline -> buffered!
        fork();
    }

    // This is tricky because the BUFFERED "A"s get duplicated by fork.
    //
    // Let's trace carefully. We label processes P (original), and children
    // C1, C2, etc.
    //
    //          BEFORE LOOP: 1 process (P), buffer = ""
    //
    //  i=0:   P does printf("A")  -> P's buffer = "A"
    //         P does fork()       -> P (buffer="A") + C1 (buffer="A")
    //         Now 2 processes, each with "A" in their buffer.
    //
    //  i=1:   P does printf("A")  -> P's buffer = "AA"
    //         P does fork()       -> P (buffer="AA") + C2 (buffer="AA")
    //
    //         C1 does printf("A") -> C1's buffer = "AA"
    //         C1 does fork()      -> C1 (buffer="AA") + C3 (buffer="AA")
    //
    //  END:   4 processes, each with "AA" in their buffer.
    //         Each flushes on exit.
    //
    //  TOTAL OUTPUT: "AA" x 4 = "AAAAAAAA" (8 A's)
    //
    //  WITHOUT the buffering issue (if we used \n or fflush):
    //    i=0: 2 processes print "A"  -> 2 A's
    //    i=1: 4 processes print "A"  -> 4 A's
    //    Total: 6 A's
    //
    //  The buffering adds 2 extra A's! The "A" from the first printf was
    //  sitting in the buffer when fork happened, so it got duplicated into
    //  all child processes and printed again when they exited.

    while (wait(nullptr) > 0) {}
    return 0;
    // Try it: compile and run, pipe to `wc -c` to count characters.
    //   ./a.out | wc -c    -> 8
    //
    // But if you run it directly in a terminal (not piped), stdout is
    // line-buffered, and "A" is not a newline, but it WILL flush on
    // process exit... the key is that piping changes the buffering mode
    // from line-buffered to fully-buffered, which changes the answer!
    //
    // Terminal (line-buffered): each printf may or may not flush
    //   depending on implementation, but exit() always flushes.
    // Piped/redirected (fully-buffered): only flushes when buffer is full
    //   or on exit(). This is where the duplication is most visible.
}
```

### How to avoid the buffering trap

```cpp
#include <cstdio>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    // Solution 1: flush before forking
    printf("hello");
    fflush(stdout);
    fork();
    // Now the buffer was empty at fork time. No duplication.

    // Solution 2: use write() instead of printf()
    // write() is unbuffered (it's a raw syscall). No userspace buffer.
    const char msg[] = "hello";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);
    fork();
    // write() goes straight to the kernel. No duplication possible.

    // Solution 3: use _exit() instead of exit() in the child
    // _exit() does NOT flush stdio buffers.
    // exit() DOES flush them (it calls fclose on all open streams).
    pid_t pid = fork();
    if (pid == 0) {
        // Child: use _exit to avoid flushing duplicated buffer contents
        _exit(0);
    }

    // Solution 4: setvbuf to disable buffering
    setvbuf(stdout, nullptr, _IONBF, 0);  // _IONBF = no buffering
    // Now printf behaves like write() -- flushes immediately.

    while (wait(nullptr) > 0) {}
    return 0;
}
```

---

### fork
- Duplicates the entire virtual address space (copy-on-write).
- File descriptors are shared (same underlying kernel objects).
- **Stdio buffers are part of the address space** -- unflushed data is duplicated.
- Use `_exit()` in children, or `fflush(stdout)` before `fork()`.
- N forks in a loop -> `2^N` processes at the end.
- printf without `\n` + fork = duplicated output (the classic trap).