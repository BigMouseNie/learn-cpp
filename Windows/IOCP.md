# IOCP

## CreateIoCompletionPort

**函数原型**

```c++
HANDLE WINAPI CreateIoCompletionPort(
  _In_     HANDLE    FileHandle,
  _In_opt_ HANDLE    ExistingCompletionPort,
  _In_     ULONG_PTR CompletionKey,
  _In_     DWORD     NumberOfConcurrentThreads
);
```

**作用**
 创建一个 I/O 完成端口（IOCP）并可将一个“**支持重叠 I/O** 的句柄”与之**关联**；之后该句柄的异步 I/O 完成会把完成包投递到此端口。

**关键参数**

- `FileHandle`：要关联的句柄（文件、管道、socket 等，**必须以重叠方式打开**），或者 `INVALID_HANDLE_VALUE` 表示仅创建端口、暂不关联。`ExistingCompletionPort`：已有的 IOCP 句柄；为 `NULL` 则创建新的 IOCP。若非空且 `FileHandle` 有效，则把该句柄**绑定**到这个已有端口。
- `CompletionKey`：**用户自定义键**，与此 `FileHandle` 绑定；之后每次该句柄的 I/O 完成都会随完成包返回这个键。通常放连接/会话上下文指针。
- `NumberOfConcurrentThreads`：此端口允许**并发**从队列取包并运行的最大线程数；为 **0** 则**默认=逻辑处理器数**。该参数仅在**创建新端口**时有效（`ExistingCompletionPort==NULL` 时生效）。

**返回值**

- 成功：返回 IOCP 句柄（若是“绑定”调用，返回的就是传入的已有端口句柄）。
- 失败：返回 `NULL`，用 `GetLastError()` 取错误。

**note**

- **一个句柄只能关联到一个 IOCP**，且一经绑定到关闭为止不可更改。
- 关联到 IOCP 的句柄**不要**再与 `ReadFileEx/WriteFileEx`（带 APC 的 Ex 版本）混用。
- 释放：IOCP 及其**所有已关联句柄**都关闭后，端口资源才会被系统回收。

------

## GetQueuedCompletionStatus

**函数原型**

```
BOOL GetQueuedCompletionStatus(
  HANDLE       CompletionPort,    // 完成端口的句柄
  LPDWORD      lpNumberOfBytes,   // [out] 完成的字节数
  PULONG_PTR   lpCompletionKey,   // [out] 当初绑定到句柄的 CompletionKey
  LPOVERLAPPED *lpOverlapped,     // [out] 对应的 OVERLAPPED 指针
  DWORD        dwMilliseconds     // 超时时间，INFINITE 表示永远等待
);

```



**作用**
 从指定 IOCP **取出一个完成包**；如当前为空，则**等待**直到有包抵达或超时。可用 `GetQueuedCompletionStatusEx` 批量取包。

**关键参数**

- `CompletionPort`：目标 IOCP 句柄。
- `lpNumberOfBytesTransferred`：返回**此次 I/O 完成的字节数**（或由 `PostQueuedCompletionStatus` 人工投递时指定的值）。
- `lpCompletionKey`：返回与该**句柄**绑定的 CompletionKey。
- `lpOverlapped`：返回当初发起 I/O 时传入的 `OVERLAPPED*`（**人工投递**时就是投的那个指针，常见地也可以是 `NULL`）。
- `dwMilliseconds`：等待超时时间；`INFINITE` 表示无限等。**超时**时函数返回 FALSE 且 `*lpOverlapped == NULL`。

**返回值与判错细节**

- **TRUE**：从端口成功取到**一个“成功的 I/O”完成包**，并把三元组写入上述三个输出参数。
- **FALSE**：两类情况——
	1. **没有取到包**（如**超时**）：`*lpOverlapped == NULL`。
	2. **取到的是“失败的 I/O”完成包**：`*lpOverlapped != NULL`，调用 `GetLastError()` 得到该 I/O 的错误码（如取消会返回 `ERROR_OPERATION_ABORTED`）。

**note**

- 调用该函数的**线程会与此 IOCP 关联**（同一线程同一时刻最多关联一个端口）。
- 如果端口在等待期间被关闭：返回 FALSE，`*lpOverlapped == NULL`，`GetLastError()==ERROR_ABANDONED_WAIT_0`。
- 可以通过在 `OVERLAPPED.hEvent` 放入一个**置低位的有效事件句柄**来**禁止**该 I/O 投递到端口（转而仅触发事件）。

> 相关：需要**一次取多个包**时用 `GetQueuedCompletionStatusEx`（可减少系统调用次数）。

------

## PostQueuedCompletionStatus

**函数原型**

```
BOOL WINAPI PostQueuedCompletionStatus(
  _In_     HANDLE       CompletionPort,
  _In_     DWORD        dwNumberOfBytesTransferred,
  _In_     ULONG_PTR    dwCompletionKey,
  _In_opt_ LPOVERLAPPED lpOverlapped
);
```

**作用**
 **手动**向 IOCP **投递一个完成包**，用来唤醒/驱动在 `GetQueuedCompletionStatus` 上等待的线程，可当作“任务队列”使用。

**关键参数与语义**

- `CompletionPort`：目标 IOCP。
- `dwNumberOfBytesTransferred / dwCompletionKey / lpOverlapped`：这**三项原封不动**地作为取包时的三元组返回；系统**不校验**其语义，`lpOverlapped` **不必**指向真实的 `OVERLAPPED`，很多场景直接传 **`NULL`** 代表“用户任务”。

**返回值**

- 成功：非零；失败：零，`GetLastError()` 取错误。

**常见用途**

- **发退出哨兵**：`PostQueuedCompletionStatus(port, 0, EXIT_KEY, NULL);`
- **派发自定义任务**：把任务指针塞进 `dwCompletionKey`，线程取出后转型处理。

## 文件异步读/写（ReadFile / WriteFile）

```
// fileapi.h, Kernel32.lib
BOOL ReadFile(
    HANDLE       hFile,
    LPVOID       lpBuffer,
    DWORD        nNumberOfBytesToRead,
    LPDWORD      lpNumberOfBytesRead,   // 重叠模式下可为 NULL
    LPOVERLAPPED lpOverlapped           // 非 NULL => 异步
);

BOOL WriteFile(
    HANDLE        hFile,
    LPCVOID       lpBuffer,
    DWORD         nNumberOfBytesToWrite,
    LPDWORD       lpNumberOfBytesWritten,   // 重叠模式下可为 NULL
    LPOVERLAPPED  lpOverlapped              // 非 NULL => 异步
);
```

- 若 `lpOverlapped != NULL` 且句柄以 `FILE_FLAG_OVERLAPPED` 打开，则为**异步**；起始偏移取自 `OVERLAPPED.Offset/OffsetHigh`（对不支持偏移的设备忽略）。完成后会向 IOCP 投递完成包。返回值：
	- TRUE：可能同步已完成（仍会向端口排包，如上所述）。
	- FALSE 且 `GetLastError()==ERROR_IO_PENDING`：表示**已成功挂起**，稍后通过 IOCP 收到完成。
	- 其他错误：立即失败。
- 取消：`CancelIoEx`；被取消的 I/O 最终以 `ERROR_OPERATION_ABORTED` 完成。

------

## Winsock 异步收发（WSARecv / WSASend）

```
int WSARecv(
  SOCKET                             s,            // 套接字
  LPWSABUF                           lpBuffers,    // 缓冲区数组
  DWORD                              dwBufferCount,// 缓冲区个数
  LPDWORD                            lpNumberOfBytesRecvd, // [out] 实际接收字节数
  LPDWORD                            lpFlags,      // 标志（MSG_PARTIAL 等）
  LPWSAOVERLAPPED                    lpOverlapped, // 异步用 OVERLAPPED
  LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine // APC 回调（IOCP下常传NULL）
);

int WSASend(
  SOCKET                             s,            // 套接字
  LPWSABUF                           lpBuffers,    // 缓冲区数组
  DWORD                              dwBufferCount,// 缓冲区个数
  LPDWORD                            lpNumberOfBytesSent, // [out] 实际发送字节数
  DWORD                              dwFlags,      // 标志（一般 0）
  LPWSAOVERLAPPED                    lpOverlapped, // 异步用 OVERLAPPED
  LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine // APC 回调，IOCP 下传 NULL
);

```

- 用法（IOCP 模式）：`lpOverlapped` 传有效的 `WSAOVERLAPPED`；`lpCompletionRoutine` 传 **NULL**，这样**完成结果走 IOCP**；若你提供了完成例程，则走 APC/告警等待，不走 IOCP。
- 返回值：
	- 0：可能同步完成（仍会向 IOCP 投递完成包）；
	- SOCKET_ERROR 且 `WSAGetLastError()==WSA_IO_PENDING`：**挂起成功**；
	- 其他错误：立即失败。
- 缓冲区与并发：I/O 未完成前，**不要动 `WSABUF`**；同一 socket 上并发调用需小心同步（常见做法：同一方向一次只挂一笔）。

------

## 异步接受连接（AcceptEx / GetAcceptExSockaddrs）

```
BOOL AcceptEx(
  SOCKET       sListenSocket,      // 监听 socket
  SOCKET       sAcceptSocket,      // 用来接收新连接的 socket
  PVOID        lpOutputBuffer,     // 输出缓冲区，保存本地/远端地址
  DWORD        dwReceiveDataLength,// 预读数据长度（可设 0）
  DWORD        dwLocalAddressLength,// 本地地址空间大小
  DWORD        dwRemoteAddressLength,// 远端地址空间大小
  LPDWORD      lpdwBytesReceived,  // [out] 实际收到的字节
  LPOVERLAPPED lpOverlapped        // 异步用 OVERLAPPED
);

```

- 作用：**一次系统调用**完成三件事：接受新连接、拿到本地/远端地址、可选地**预读**一段客户端发来的数据（`dwReceiveDataLength`）。非常适合 IOCP。

- 关键要点：

	1. `sAccept` 要提前用 `WSA_FLAG_OVERLAPPED` 创建，协议族/类型与 `sListen` 一致；

	2. `lpOutputBuffer` 需要能同时容纳“预读数据 + 两个地址结构”；地址部分每个需要 `sizeof(SOCKADDR_IN6)+16`（IPv4 用 `SOCKADDR_IN`）。

	3. 完成后**必须**调用

		```
		setsockopt(sAccept, SOL_SOCKET, SO_UPDATE_ACCEPT_CONTEXT,
		           (char*)&sListen, sizeof(sListen));
		```

		否则后续 `getpeername/getsockname` 等可能异常，资源回收也不完整。

	4. 完成后，把 `sAccept` **关联到 IOCP**（`CreateIoCompletionPort`），然后对其投递 `WSARecv` 等。

- 解析地址：

```
VOID GetAcceptExSockaddrs(
    PVOID                lpOutputBuffer,
    DWORD                dwReceiveDataLength,
    DWORD                dwLocalAddressLength,
    DWORD                dwRemoteAddressLength,
    sockaddr**           LocalSockaddr,
    LPINT                LocalSockaddrLength,
    sockaddr**           RemoteSockaddr,
    LPINT                RemoteSockaddrLength
);
```

把 `AcceptEx` 的 `lpOutputBuffer` 切分出本地/远端地址指针和长度。

> 备注：`AcceptEx`/`GetAcceptExSockaddrs` 在头文件 **mswsock.h**，库 **Mswsock.lib** 中；也可按历史做法用 `WSAIoctl(SIO_GET_EXTENSION_FUNCTION_POINTER)` 在运行时获取函数指针。

------

## 与 IOCP 的协作要点

- 句柄必须支持**重叠 I/O**（文件以 `FILE_FLAG_OVERLAPPED` 打开；socket 用 `WSA_FLAG_OVERLAPPED` 创建），并且**关联到 IOCP**。之后对该句柄发起的每个重叠 I/O 完成，都会向端口排入完成包，供 `GetQueuedCompletionStatus(Ex)` 取回。
- `lpOverlapped` 返回的就是你当初投递时传入的那个地址（通常嵌在你自定义的 per-IO 结构里）；在 GQCS 里把它还原成自定义对象即可继续处理。
- 取消与关闭：被 `CancelIoEx`/`closesocket` 取消的 I/O 最终也会排入完成队列，错误码为 `ERROR_OPERATION_ABORTED`（或对应 Winsock 错误）；一定要在工作线程里**消费**这些完成包以清理资源。



## Note

- **句柄重叠**：与 IOCP 绑定的句柄必须支持 overlapped I/O（创建文件时用 `FILE_FLAG_OVERLAPPED`；Winsock 使用重叠 API）。
- **CompletionKey 设计**：通常为**对象/连接上下文指针**，一把就能定位到会话结构。
- **OVERLAPPED 生命周期**：发起 I/O 后，**直到你从 IOCP 取到对应完成包**（或取消并取到取消完成）之前，`OVERLAPPED` 所在内存不得释放/复用。
- **取消 I/O**：`CancelIoEx` 可能使完成包以**失败**到达（如 `ERROR_OPERATION_ABORTED`）；按上文的“返回值判错细节”处理。
- **并发度**：`NumberOfConcurrentThreads` 设为 0 或≈CPU 数即可，避免过度并发导致抢占/切换。

如果你需要，我可以给你一段**最小可运行**的示例骨架（只含原型调用与判错，便于拷贝到工程里）。