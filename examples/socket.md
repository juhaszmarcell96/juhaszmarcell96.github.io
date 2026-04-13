# Sockets

---

## Core concepts

A socket is a file descriptor that represents one endpoint of a network communication. The workflow differs significantly between TCP (connection-oriented, reliable, stream) and UDP (connectionless, unreliable, datagram).

The key system calls all live in `<sys/socket.h>`, `<netinet/in.h>`, `<arpa/inet.h>`, and `<unistd.h>`.

---

## TCP workflow

TCP requires a connection to be established before data flows. The server listens and accepts; the client connects.

```
Server                              Client
------                              ------
socket()                            socket()
bind()                              
listen()                            
                                    connect()  ---SYN--->
accept()  <---handshake--->         
recv() / send()                     send() / recv()
close()                             close()
```

### TCP server

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstdio>
#include <cstdint>
#include <cstring>

int main()
{
    // 1. create socket
    //    AF_INET     = IPv4
    //    SOCK_STREAM = TCP
    std::int32_t server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0)
    {
        std::perror("socket");
        return 1;
    }

    // allow port reuse so you don't get "address already in use" on restart
    std::int32_t opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 2. bind to an address + port
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;  // listen on all interfaces
    addr.sin_port = htons(8080);        // network byte order

    if (bind(server_fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0)
    {
        std::perror("bind");
        close(server_fd);
        return 1;
    }

    // 3. listen -- backlog = max queued connections
    if (listen(server_fd, 128) < 0)
    {
        std::perror("listen");
        close(server_fd);
        return 1;
    }

    std::printf("server listening on port 8080\n");

    // 4. accept -- blocks until a client connects
    //    returns a NEW file descriptor for this specific connection
    sockaddr_in client_addr{};
    socklen_t client_len = sizeof(client_addr);
    std::int32_t client_fd = accept(
        server_fd,
        reinterpret_cast<sockaddr*>(&client_addr),
        &client_len
    );

    if (client_fd < 0)
    {
        std::perror("accept");
        close(server_fd);
        return 1;
    }

    char client_ip[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, sizeof(client_ip));
    std::printf("client connected from %s:%d\n", client_ip, ntohs(client_addr.sin_port));

    // 5. communicate
    char buffer[1024];
    ssize_t bytes_read = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
    if (bytes_read > 0)
    {
        buffer[bytes_read] = '\0';
        std::printf("received: %s\n", buffer);

        const char* response = "hello from server";
        send(client_fd, response, std::strlen(response), 0);
    }

    // 6. close both file descriptors
    close(client_fd);  // the connection
    close(server_fd);  // the listening socket

    return 0;
}
```

### TCP client

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstdio>
#include <cstdint>
#include <cstring>

int main()
{
    // 1. create socket
    std::int32_t sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (sock_fd < 0)
    {
        std::perror("socket");
        return 1;
    }

    // 2. specify server address
    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8080);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

    // 3. connect -- initiates the TCP three-way handshake
    //    no need to bind() -- the OS picks an ephemeral source port
    if (connect(sock_fd, reinterpret_cast<sockaddr*>(&server_addr), sizeof(server_addr)) < 0)
    {
        std::perror("connect");
        close(sock_fd);
        return 1;
    }

    std::printf("connected to server\n");

    // 4. communicate
    const char* message = "hello from client";
    send(sock_fd, message, std::strlen(message), 0);

    char buffer[1024];
    ssize_t bytes_read = recv(sock_fd, buffer, sizeof(buffer) - 1, 0);
    if (bytes_read > 0)
    {
        buffer[bytes_read] = '\0';
        std::printf("server replied: %s\n", buffer);
    }

    // 5. close
    close(sock_fd);

    return 0;
}
```

### Important TCP details

**`recv` does not guarantee you get the full message in one call.** TCP is a byte stream - the kernel can deliver data in any chunk size. In production you need a receive loop:

```cpp
ssize_t recv_all(std::int32_t fd, char* buffer, std::size_t expected)
{
    std::size_t total = 0;
    while (total < expected)
    {
        ssize_t n = recv(fd, buffer + total, expected - total, 0);
        if (n <= 0)
        {
            // 0 = peer closed, -1 = error
            return n;
        }
        total += static_cast<std::size_t>(n);
    }
    return static_cast<ssize_t>(total);
}
```

**`accept` returns a new fd** - the original `server_fd` keeps listening. A real server would loop on `accept` and handle each `client_fd` in a separate thread or with `epoll`/`select`.

---

## UDP workflow

UDP has no connection. Each `sendto`/`recvfrom` is an independent datagram. No handshake, no guaranteed delivery, no ordering.

```
Server                              Client
------                              ------
socket()                            socket()
bind()                              
recvfrom()  <----datagram----       sendto()
sendto()    ----datagram---->       recvfrom()
close()                             close()
```

### UDP server

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstdio>
#include <cstdint>
#include <cstring>

int main()
{
    // 1. create socket -- SOCK_DGRAM = UDP
    std::int32_t sock_fd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock_fd < 0)
    {
        std::perror("socket");
        return 1;
    }

    // 2. bind to port
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(9090);

    if (bind(sock_fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0)
    {
        std::perror("bind");
        close(sock_fd);
        return 1;
    }

    std::printf("udp server listening on port 9090\n");

    // 3. receive -- no listen() or accept() needed
    //    recvfrom tells us WHO sent the datagram
    char buffer[1024];
    sockaddr_in client_addr{};
    socklen_t client_len = sizeof(client_addr);

    ssize_t bytes_read = recvfrom(
        sock_fd,
        buffer,
        sizeof(buffer) - 1,
        0,
        reinterpret_cast<sockaddr*>(&client_addr),
        &client_len
    );

    if (bytes_read > 0)
    {
        buffer[bytes_read] = '\0';

        char client_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, sizeof(client_ip));
        std::printf("received from %s:%d -> %s\n",
            client_ip, ntohs(client_addr.sin_port), buffer);

        // 4. reply to whoever sent it
        const char* response = "ack";
        sendto(
            sock_fd,
            response,
            std::strlen(response),
            0,
            reinterpret_cast<sockaddr*>(&client_addr),
            client_len
        );
    }

    close(sock_fd);
    return 0;
}
```

### UDP client

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstdio>
#include <cstdint>
#include <cstring>

int main()
{
    // 1. create socket
    std::int32_t sock_fd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock_fd < 0)
    {
        std::perror("socket");
        return 1;
    }

    // 2. specify destination -- no connect() needed
    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(9090);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

    // 3. send datagram
    const char* message = "hello udp";
    sendto(
        sock_fd,
        message,
        std::strlen(message),
        0,
        reinterpret_cast<sockaddr*>(&server_addr),
        sizeof(server_addr)
    );

    // 4. receive reply
    char buffer[1024];
    sockaddr_in from_addr{};
    socklen_t from_len = sizeof(from_addr);

    ssize_t bytes_read = recvfrom(
        sock_fd,
        buffer,
        sizeof(buffer) - 1,
        0,
        reinterpret_cast<sockaddr*>(&from_addr),
        &from_len
    );

    if (bytes_read > 0)
    {
        buffer[bytes_read] = '\0';
        std::printf("server replied: %s\n", buffer);
    }

    close(sock_fd);
    return 0;
}
```

### Note: `connect()` on a UDP socket

You *can* call `connect()` on a UDP socket. It doesn't establish a connection - it just sets a default destination so you can use `send()`/`recv()` instead of `sendto()`/`recvfrom()`, and the kernel will filter out datagrams from other sources:

```cpp
// "connected" udp -- still no handshake
connect(sock_fd, reinterpret_cast<sockaddr*>(&server_addr), sizeof(server_addr));
send(sock_fd, message, std::strlen(message), 0);      // no need for sendto
recv(sock_fd, buffer, sizeof(buffer), 0);              // no need for recvfrom
```

---

## TCP vs UDP side-by-side

| Aspect | TCP | UDP |
|---|---|---|
| Socket type | `SOCK_STREAM` | `SOCK_DGRAM` |
| Connection | `connect()` + `accept()` handshake | none (or optional `connect()` for convenience) |
| Server setup | `bind` -> `listen` -> `accept` | `bind` -> `recvfrom` |
| Send/receive | `send()` / `recv()` | `sendto()` / `recvfrom()` |
| Message boundaries | no - byte stream, you must frame | yes - one `sendto` = one `recvfrom` |
| Ordering | guaranteed | not guaranteed |
| Reliability | retransmits lost data | fire and forget |
| Use case | HTTP, database connections, file transfer | DNS, game state updates, video streaming |

---

## Common pitfalls

**Byte order**: `htons()` / `htonl()` convert host byte order to network byte order (big-endian). Forgetting this means your port number is garbled on little-endian machines (which is most of them).

**`SIGPIPE`**: writing to a closed TCP socket sends `SIGPIPE` which kills your process by default. Common fix: `signal(SIGPIPE, SIG_IGN)` or use `MSG_NOSIGNAL` flag with `send()`.

**File descriptor leaks**: every `accept()` returns a new fd. If you forget to `close()` it, you leak until you hit the process fd limit.

**Blocking vs non-blocking**: all these examples use blocking I/O. In production you typically use `epoll` (Linux), `kqueue` (macOS), or `select`/`poll` to multiplex many connections on one thread, or combine with a thread pool.