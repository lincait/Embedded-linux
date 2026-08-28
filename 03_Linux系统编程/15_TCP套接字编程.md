# TCP 套接字编程

# 套接字是什么

套接字（Socket）是网络通信的编程接口。它封装了 TCP/IP 协议的细节，让开发者不用了解底层协议栈就能收发数据。英文直译为"插座"——进程把自己的 IP 和端口"插"在套接字上，另一端的进程就能通过这个接口找到它并交换数据。

socket 本质上可以理解成：Linux 内核中的一个“网络通信对象”，里面保存了这个通信端点相关的状态，程序里的 `socketfd` 就是指向这个内核对象的文件描述符。

一个套接字由三个属性组成：

- **网络地址**（IP）：标识网络上的设备。
- **端口号**：16 位数字（0~65535），标识设备上的特定应用。低于 1024 的是特权端口，只有特权进程能绑定。
- **协议**：TCP 或 UDP，定义数据传输的规则。

根据传输方式不同，套接字分两种：

- **流套接字（SOCK_STREAM）**：基于 TCP，面向连接、可靠，数据像流水一样按顺序到达，如网页服务器。
- **数据报套接字（SOCK_DGRAM）**：基于 UDP，无连接，每个报文独立传输，可能丢失或乱序，如在线视频会议。

套接字本质上也是文件描述符：底层同样存放在 `struct file` 结构体中，socket 相关数据存在该结构体的私有数据字段里。所以对套接字的操作和文件操作是同一套系统调用思路，最后也用 `close` 关闭。

# 地址结构体

`bind`、`connect`、`accept` 等函数都需要一个指向 `struct sockaddr *` 的地址参数。`struct sockaddr` 是通用基类，实际使用中根据地址族用对应的具体结构体：

```c
struct sockaddr {
    sa_family_t sa_family;  // 地址族，如 AF_INET、AF_INET6、AF_UNIX
    char        sa_data[14];// 具体地址数据，布局取决于地址族
};
```

IPv4 对应的具体结构体是 `struct sockaddr_in`：

```c
struct sockaddr_in {
    sa_family_t    sin_family;  // 地址族，填 AF_INET
    in_port_t      sin_port;    // 端口号，网络字节序
    struct in_addr sin_addr;    // IP 地址，网络字节序
};

struct in_addr {
    uint32_t s_addr;  // 网络字节序中的地址
};
```

要点：

- 设置时把 `sockaddr_in` 强制转换成 `struct sockaddr *` 传入，编译器才不会警告。
- `sin_port` 和 `sin_addr` 都必须是**网络字节序**，用 `htons`/`htonl` 转换。
- 绑定任意本机地址用 `INADDR_ANY`（即 0.0.0.0），配合 `htonl` 使用，表示监听本机所有网卡。
- `socklen_t` 是 `unsigned int` 的别名，表示地址结构体的字节大小。
- 每次使用前先 `memset` 清零，避免残留数据。

# 常用函数

## socket：创建套接字

```c
int socket(int domain, int type, int protocol);
```

- `domain`：通信域（地址族）。IPv4 用 `AF_INET`，本机进程间通信用 `AF_UNIX`。
- `type`：套接字类型。`SOCK_STREAM`（流，TCP）或 `SOCK_DGRAM`（数据报，UDP）。
- `protocol`：协议，填 0 表示按类型自动选择默认协议（SOCK_STREAM 默认 TCP）。
- 返回一个文件描述符（fd），失败返回 -1。

## bind：绑定地址

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

`socket` 创建出来的套接字只是存在于某个地址族里，还没有地址和端口。`bind` 把 `addr` 指定的地址（IP + 端口）绑定到 sockfd 上，传统上叫"给套接字分配一个名称"。服务端必须调用，客户端一般不调用（由内核自动分配临时端口）。成功返回 0，失败返回 -1。

## listen：进入监听状态

```c
int listen(int sockfd, int backlog);
```

把套接字从**主动套接字**变成**被动套接字**，表明它只用来接受连接，不再主动发起连接。`backlog` 是等待队列的最大长度——还没被 `accept` 处理的已完成连接可以先排队等在这里。成功返回 0，失败返回 -1。

## accept：接受连接

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

从监听套接字的待处理队列中取出第一个连接请求，**创建一个新的连接套接字**并返回它的 fd。原始监听套接字不受影响，可以继续 accept 下一个连接。`addr` 用于返回客户端的地址，不关心就填 NULL。失败返回 -1。

注意：之后收发数据用的是 accept 返回的新 fd，而不是监听 fd。

当前没有已经建立好的客户端连接时会阻塞。

## connect：发起连接

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

客户端调用，向服务端发起连接，内部会触发三次握手。`addr` 是服务端地址。成功返回 0，失败返回 -1。`connect` 调用成功即代表连接已建立。

## send / recv：发送和接收数据

```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

- `buf` 是用户自己维护的缓冲区（不是内核的），`len` 是要发送/接收的字节数。
- `flags` 一般填 0。`send` 可用 `MSG_NOSIGNAL` 避免对端关闭时产生 SIGPIPE 信号；`recv` 可用 `MSG_PEEK` 只读不取走数据。
- `send` 返回实际发送的字节数；`recv` 返回实际接收的字节数，**返回 0 表示对端正常关闭了连接**，返回 -1 表示出错。

## close / shutdown：关闭

```c
int close(int fd);
int shutdown(int sockfd, int how);
```

- `close` 使底层文件描述符的引用计数减一，减为 0 才真正释放套接字资源。TCP 通信中，双方都 close 才会触发完整的四次挥手。
- `shutdown` 可以只关闭一个方向：
  - `SHUT_RD`：关闭读，之后不再接收数据，阻塞在 recv 的调用返回 0。
  - `SHUT_WR`：关闭写，向对端发送 FIN，对端的 recv 会返回 0。
  - `SHUT_RDWR`：同时关闭读写。

# TCP 服务端编程流程

1. `socket()` 创建监听套接字（`AF_INET` + `SOCK_STREAM`），获得用于监听的fd。
2. 填写 `sockaddr_in`（地址族、IP、端口，注意字节序），`bind()` 绑定地址。
3. `listen()` 进入监听状态。
4. `accept()` 阻塞等待客户端连接，拿到新的连接的客户端套接字 fd，用于和对应客户端通信。
5. 用这个新 fd `send`/`recv` 收发数据。
6. 通信结束 `close()` 关闭连接套接字（和监听套接字）。

# TCP 客户端编程流程

1. `socket()` 创建套接字。
2. 填写服务端的 `sockaddr_in`，`connect()` 发起连接（内部完成三次握手）。
3. 连接建立后直接 `send`/`recv` 收发数据。
4. 通信结束 `close()` 关闭套接字。

对比可见：服务端多出 `bind` + `listen` + `accept` 三步，客户端则是一步 `connect` 到位。

# 缓冲区的补充说明

## 进程的 I/O 缓冲区与内核网络缓冲区

进程的 I/O 缓冲区是标准库为用户维护的，用于文件读写或终端交互，分为输入缓冲区和输出缓冲区。而网络编程中，内核会在内核空间维护网络缓冲区，同样分输入输出：

- **数据接收**：数据到达网卡后，驱动先把数据放进内核的接收缓冲区，应用再用 `recv`/`read` 取走。
- **数据发送**：应用用 `send`/`write` 把数据交给内核的发送缓冲区，内核再按协议适时发到网络上。

网络缓冲区由内核管理，用户无法干预；用户可以在自己的堆空间里再构建额外的应用层缓冲区。

## 进程的缓冲模式

- **行缓冲**：碰到换行符或缓冲区满才刷写。标准输出连终端时默认行缓冲，保证每行输入立即看到输出。
- **全缓冲**：缓冲区满才一次性刷写。写文件默认是全缓冲，减少系统调用次数。
- **无缓冲**：数据直接刷写。标准错误（stderr）默认无缓冲，保证错误信息立即输出。

用 `setvbuf()` 可以设置流的缓冲模式，`fflush()` 可以手动刷写。网络编程中要注意：printf 这类行缓冲输出如果没刷写，可能不会及时出现在屏幕上——调试时要么加 `\n`，要么 `fflush(stdout)`。

# 支持多连接的服务器

单个 accept 只能处理一个连接。实际服务器要同时服务多个客户端，思路有两种：

- **多线程**：主线程 accept 到新连接后，创建一个子线程专门处理这个连接的收发，主线程立刻回去继续 accept。
- **多进程**：同上，用 `fork` 创建子进程处理连接，父进程继续 accept。

两种方式的要点都是：**处理连接的代码和接受连接的代码分离**，accept 循环不能阻塞在某个连接的收发上。这也呼应了"进程间通信"章节里管道、共享内存等多进程并发的内容。
