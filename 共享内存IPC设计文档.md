# Linux 共享内存 IPC 设计文档

**版本**: 2.2 (Stream Model + Sync Support + Full Detail)
**日期**: 2025-12-09
**状态**: 完善中

## 1. 引言

### 1.1 背景
为了提供更灵活、通用且符合现代网络编程习惯的 IPC 机制，我们将设计从“消息整包传输”重构为“流式传输 (Byte Stream)”。这使得共享内存的使用体验极其接近 TCP Socket：支持 `read`/`write` 的部分读写，且提供 File Descriptor (FD) 以支持 `epoll`/`select` 等 IO 多路复用机制。

### 1.2 目标
- **Socket-like 体验**: 提供类似 `read` / `write` 的接口，支持部分读写。
- **多客户端支持**: 采用“控制流”与“数据流”分离的设计，支持多 Client 并发连接。
- **双模支持**:
    - **同步/阻塞 (Blocking)**: 默认模式，简单易用，适合简单逻辑。
    - **异步/非阻塞 (Non-Blocking)**: 配合 `epoll`，适合高并发/事件驱动场景。
- **高性能**: 基于共享内存 RingBuffer，维持零拷贝（或单次拷贝）的优势。

## 2. 总体架构

### 2.1 双层通信模型
系统分为**控制平面**和**数据平面**：

1.  **控制平面 (Control Plane)**:
    -   **介质**: Unix Domain Socket (UDS).
    -   **作用**: 负责连接握手、身份鉴权、协商共享内存参数（名称、大小）。
    -   **流程**: Client 连接 Server 的 UDS -> Server 分配唯一 Session ID -> 双方建立专属 SHM 通道。

2.  **数据平面 (Data Plane)**:
    -   **介质**: 共享内存 (SHM) + Named FIFO.
    -   **作用**: 负责高频、大数据量的业务数据传输。
    -   **特点**: 1对1 独占通道，无锁竞争（SPSC模型）。

### 2.2 内存布局 (Data Plane)
采用原子操作保护索引，配合 **Named FIFO (管道)** 进行唤醒通知。

```text
+-------------------------+ <--- 共享内存起始地址
|   Control Header        |
| (Atomic Indexes, Sizes) |
+-------------------------+ <--- 64字节对齐
|                         |
|      Ring Buffer        |
|      (Raw Bytes)        |
|                         |
+-------------------------+
```

### 2.3 通信与通知模型
- **数据通道**: 共享内存 RingBuffer。
- **通知通道**: **Named Pipe (FIFO)**。
    - 每个 Session 包含两个 FIFO（路径包含 Session ID）：
        - `signal_read_fd`: 消费者监听。生产者写数据后写入此管道 -> 唤醒消费者。
        - `signal_write_fd`: 生产者监听。消费者读数据后写入此管道 -> 唤醒生产者。

## 3. 详细设计

### 3.1 核心数据结构

```c
#include <stdatomic.h>
#include <stdint.h>

// 64字节对齐宏，避免 False Sharing
#define CACHE_LINE_SIZE 64
#define ALIGNED_CACHE __attribute__((aligned(CACHE_LINE_SIZE)))

typedef struct {
    uint32_t magic;             // 魔数 (e.g. 0x58495043 'XIPC')
    uint32_t version;           // 版本号
    uint32_t capacity;          // 数据区容量
    uint32_t head_offset;       // 数据区起始偏移量
    
    // 读写索引 (原子变量，64位防止溢出)
    // 必须使用 atomic_load_explicit / atomic_store_explicit 配合 memory_order
    atomic_uint_fast64_t read_idx;  ALIGNED_CACHE; // 读指针
    atomic_uint_fast64_t write_idx; ALIGNED_CACHE; // 写指针
} shm_header_t;
```

### 3.2 连接握手协议 (Handshake Protocol)
在调用 `ipc_socket_open` 之前，Client 和 Server 需通过 Unix Domain Socket 完成以下协商：

1.  **Connect**: Client 连接 Server 的 UDS (e.g. `/tmp/xipc_control.sock`).
2.  **Request**: Client 发送握手请求，包含：
    - Client UID (鉴权)
    - **Requested Size** (期望的共享内存大小)
3.  **Decision**: Server 收到请求后，根据业务逻辑决定：
    - 是否允许连接
    - 分配的 Session ID
    - 最终确定的 SHM 大小 (Actual Size)
4.  **Response**: Server 返回 Session ID 和 Actual Size。
5.  **Attach**: 双方使用 Session ID 作为 `name` 调用 `ipc_socket_open`。

### 3.3 控制平面 API 设计 (Control Plane)

为了支持灵活的协商，API 将请求接收与响应决策分离：

```c
// 客户端请求对象
typedef struct {
    void* _internal_handle; // 内部句柄 (保存连接状态)
    int client_uid;         // 客户端 UID
    size_t requested_size;  // 客户端期望的大小
} ipc_request_t;

// 会话结果信息
typedef struct {
    char session_name[64]; // 协商后的 SHM 名称
    size_t shm_size;       // 最终确认的大小
} ipc_session_info_t;

// Server 端控制句柄
typedef struct ipc_control_server ipc_control_server_t;

/**
 * 创建控制平面监听服务
 * @param address 监听地址 (e.g. "/tmp/xipc_ctrl.sock")
 * @return 控制句柄，失败返回 NULL
 */
ipc_control_server_t* ipc_control_server_create(const char* address);

/**
 * 接受客户端连接请求
 * 阻塞等待，直到收到一个客户端的握手请求。
 * 注意：此时尚未给客户端回复，服务端需根据 req 内容决定如何响应。
 * 
 * @param server 控制句柄
 * @param out_req 输出参数，填充客户端的请求信息
 * @return 0 成功, -1 失败
 */
int ipc_control_server_accept_request(ipc_control_server_t* server, ipc_request_t* out_req);

/**
 * 发送握手响应 (同意连接)
 * 服务端决定分配的 session_name 和 actual_size 后调用此函数。
 * 
 * 注意：为了防止数据丢失，函数内部应执行“优雅关闭 (Graceful Shutdown)”流程：
 * 1. 发送响应数据。
 * 2. shutdown(SHUT_WR) 关闭写端。
 * 3. (可选) 阻塞读取直到收到 EOF (客户端关闭连接)。
 * 4. close() 释放资源。
 * 
 * @param server 控制句柄
 * @param req 对应的请求对象
 * @param session_name 分配的会话名称
 * @param actual_size 最终分配的大小
 * @return 0 成功, -1 失败
 */
int ipc_control_server_respond(ipc_control_server_t* server, ipc_request_t* req, 
                             const char* session_name, size_t actual_size);

/**
 * 拒绝连接
 * @param reason 错误码
 */
void ipc_control_server_reject(ipc_control_server_t* server, ipc_request_t* req, int reason);

/**
 * 客户端连接控制平面并执行握手
 * 
 * @param address 服务端地址
 * @param requested_size 期望申请的共享内存大小
 * @param out_info 输出参数，接收服务端最终分配的会话信息
 * @return 0 成功, -1 失败
 */
int ipc_control_client_connect(const char* address, size_t requested_size, ipc_session_info_t* out_info);

/**
 * 销毁控制句柄
 */
void ipc_control_server_destroy(ipc_control_server_t* server);
```

### 3.4 数据平面 API 设计 (Data Plane)

```c
typedef struct ipc_socket {
    // 基础信息
    char name[64];           // Socket 名称 (Session ID)
    int is_server;           // 1=Server (Creator), 0=Client (Attacher)
    int flags;               // 标志位 (IPC_NONBLOCK)

    // 共享内存资源
    int shm_fd;              // SHM 文件描述符
    void* shm_base;          // SHM 映射基地址
    size_t shm_size;         // 映射大小
    shm_header_t* header;    // 便捷指针：指向头部
    uint8_t* data_buffer;    // 便捷指针：指向 RingBuffer 数据区

    // 通知通道 (FIFO) - 用于 epoll
    int my_event_fd;         // 我监听的 FD (Read End)
    int peer_event_fd;       // 对方监听的 FD (Write End)
} ipc_socket_t;

// 标志位
#define IPC_NONBLOCK 0x01 // 非阻塞模式
#define IPC_CREAT    0x02 // 创建模式 (Server)

/**
 * 打开/创建 IPC 通道
 * @param name 唯一的会话名称 (由握手阶段协商得出)
 * @param size 共享内存大小 (仅 IPC_CREAT 时有效)
 * @param flags IPC_NONBLOCK | IPC_CREAT
 * @return 句柄指针，失败返回 NULL
 */
ipc_socket_t* ipc_socket_open(const char* name, int size, int flags);

/**
 * 设置/清除阻塞模式
 * 类似于 fcntl(fd, F_SETFL, O_NONBLOCK)
 * @param nonblock 1=非阻塞, 0=阻塞
 */
int ipc_set_nonblock(ipc_socket_t* sock, int nonblock);

/**
 * 获取用于 epoll 的 Event FD
 * @return fd
 */
int ipc_get_event_fd(ipc_socket_t* sock);

/**
 * 查询当前可读字节数
 */
int ipc_readable_bytes(ipc_socket_t* sock);

/**
 * 查询当前可写字节数
 */
int ipc_writable_bytes(ipc_socket_t* sock);

/**
 * 发送数据
 * 阻塞模式: 缓冲区满则等待，直到全部写入或出错。
 * 非阻塞模式: 缓冲区满立即返回 -1 (EAGAIN) 或写入部分数据。
 * @return 实际写入字节数
 */
ssize_t ipc_write(ipc_socket_t* sock, const void* data, size_t len);

/**
 * 接收数据
 * 阻塞模式: 缓冲区空则等待，直到读取到至少 1 字节。
 * 非阻塞模式: 缓冲区空立即返回 -1 (EAGAIN)。
 * @return 实际读取字节数
 */
ssize_t ipc_read(ipc_socket_t* sock, void* buf, size_t len);

/**
 * 关闭并清理资源
 * @param unlink 是否删除共享内存文件和FIFO文件
 */
void ipc_close(ipc_socket_t* sock, int unlink);
```

## 4. 关键逻辑流程

### 4.1 发送流程 (ipc_write)

1. **计算空间**: `space = capacity - (write - read)`.
2. **非阻塞检查**:
   - 如果 `space == 0` 且是 `NONBLOCK`: 返回 -1, `errno = EAGAIN`.
3. **阻塞等待**:
   - 如果 `space == 0` 且是 `BLOCKING`: 
     - 循环调用 `read(signal_write_fd, ...)` 阻塞等待消费者腾出空间。
     - *注意*: 这里利用 OS 的 Pipe 阻塞机制挂起线程，避免 CPU 空转。
     - 被唤醒后，重新计算 `space`。
4. **数据拷贝**:
   - `chunk = min(len, space)`.
   - `memcpy` 数据到 RingBuffer (处理 RingBuffer 回绕)。
   - `atomic_add(&write_idx, chunk)`.
5. **触发通知**:
   - 写入 `signal_read_fd` (1字节)，通知对端有数据可读。
   - *优化*: 仅当 `old_write_idx == read_idx` (从空变有) 时才必须通知，或者为了简化逻辑总是通知(Level Trigger通过读取清理)。
6. **循环处理**:
   - 如果是 `BLOCKING` 且 `chunk < len` (未写完): `len -= chunk`, `data += chunk`, 重复步骤 1 继续写剩余部分。
   - 否则返回 `total_written`.

### 4.2 接收流程 (ipc_read)

1. **计算数据**: `avail = write - read`.
2. **非阻塞检查**:
   - 如果 `avail == 0` 且是 `NONBLOCK`: 返回 -1, `errno = EAGAIN`.
3. **阻塞等待**:
   - 如果 `avail == 0` 且是 `BLOCKING`:
     - 循环调用 `read(signal_read_fd, ...)` 阻塞等待生产者写入数据。
4. **数据拷贝**:
   - `chunk = min(len, avail)`.
   - `memcpy` 数据到用户 Buffer.
   - `atomic_add(&read_idx, chunk)`.
5. **触发通知**:
   - 写入 `signal_write_fd` (1字节)，通知对端有空间了。
6. 返回 `chunk`.

### 4.3 初始化流程 (ipc_socket_open)
1. `shm_open()`: 打开 SHM 对象。
2. `ftruncate()`: 设置大小 (仅 Server)。
3. `mmap()`: 映射 SHM。
4. `mkfifo()`: 创建两个 Named Pipe (e.g. `name_s2c`, `name_c2s`).
5. `open()`: 以 `O_RDWR` (避免 pipe 无 reader 时 open 阻塞) 打开 Pipe.

## 5. 使用示例 (多客户端模式)

### 5.1 Server 端 (Control + Data)

```c
// 1. 启动 Control Plane
ipc_control_server_t* ctrl_srv = ipc_control_server_create("/tmp/xipc_ctrl.sock");
if (!ctrl_srv) exit(1);

int epfd = epoll_create(1);
// ... 将 ctrl_srv 内部的 fd 加入 epoll ...

while(1) {
    // 2. 接受握手请求
    ipc_request_t req;
    if (ipc_control_server_accept_request(ctrl_srv, &req) == 0) {
        printf("Request from UID: %d, Size: %zu\n", req.client_uid, req.requested_size);
        
        // 3. 决策: 生成 Session ID 并确定大小 (可根据 req.requested_size 调整)
        char session_name[64];
        static int sess_id = 0;
        sprintf(session_name, "/sess_%d", ++sess_id);
        size_t final_size = (req.requested_size > 0) ? req.requested_size : 1024*1024;

        // 4. 创建数据通道 (SHM + FIFO)
        ipc_socket_t* shm_sock = ipc_socket_open(session_name, final_size, IPC_CREAT | IPC_NONBLOCK);
        
        if (shm_sock) {
            // 5. 发送响应给 Client
            ipc_control_server_respond(ctrl_srv, &req, session_name, final_size);
            
            // 6. 将 shm_sock 加入 Epoll 监听数据
            struct epoll_event ev;
            ev.events = EPOLLIN;
            ev.data.ptr = shm_sock;
            epoll_ctl(epfd, EPOLL_CTL_ADD, ipc_get_event_fd(shm_sock), &ev);
        } else {
            ipc_control_server_reject(ctrl_srv, &req, 500);
        }
    }
    
    // 处理数据事件...
}
```

### 5.2 Client 端

```c
// 1. 连接 Control Plane 并发起协商
ipc_session_info_t info;
// 申请 2MB 空间
if (ipc_control_client_connect("/tmp/xipc_ctrl.sock", 2*1024*1024, &info) != 0) {
    perror("Connect failed");
    exit(1);
}

printf("Negotiated Session: %s, Size: %zu\n", info.session_name, info.shm_size);

// 2. 连接数据通道
ipc_socket_t* shm_sock = ipc_socket_open(info.session_name, 0, IPC_NONBLOCK);
if (!shm_sock) exit(1);

// ... 读写数据 ...
```

## 附录 A: 方案对比 (消息式 vs 流式)

### 方案 1: 消息式 (Message Protocol) - 旧方案
- **特点**: 库内部处理分片、重组；接收端总是收到完整的“包”。
- **优点**: 
  - 应用层简单，无需处理粘包/半包。
  - 适合RPC类的“一问一答”短消息场景。
- **缺点**: 
  - **复杂性高**: 库内部需要维护分片状态机，容易出错（尤其在多生产者场景）。
  - **灵活性差**: 难以像TCP那样流式处理超大数据（不用等全收完再处理）。
  - **Epoll集成难**: 很难在不引入额外套接字对的情况下，实现精确的“消息到达”通知。

### 方案 2: 流式 (Stream/Socket-like) - 新方案
- **特点**: 提供字节流读写；应用层负责封包（如 Length-Prefix）；支持 Epoll。
- **优点**: 
  - **通用性强**: 语义与 TCP Socket 一致，现有网络库代码容易移植。
  - **生态集成**: 通过 Named Pipe 暴露 FD，可完美融入 Nginx/Redis/Libevent 等事件循环。
  - **核心简单**: 库只需关注 RingBuffer 的搬运和通知，Bug率更低。
  - **双模支持**: 既支持简单脚本的同步调用，也支持高性能服务的异步调用。
- **缺点**: 
  - **门槛稍高**: 应用层必须自己处理协议 framing（如先读4字节长度，再读Body）。
