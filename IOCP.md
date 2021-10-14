# IOCP 使用

[url](https://developer.aliyun.com/article/708589)

#### SOCKET

用于IOCP的socket必须是由WSASocket来创建的异步套接字，具体的函数为

```c++
SOCKET WSASocket(
		int af,
  	int type,
  	int protocol,
  	LPWSAPROTOCOL_INFO lpProtocolInfo,  // 一个指向PROTOCOL_INFO的指针，非空则忽略前3个字段
  	GROUP g,
  	DWORD dwFlags				// 属性表述
);
```

一般我们这样创建一个TCP异步套接字:

```c++
SOCKET socket = WSASocket(
		AF_INET,				// AF协议簇
  	SOCK_STREAM,		// 流协议
  	0,							// 为0则默认选择，通过前两个会被定义为TCPsocket
  	NULL,
  	0,
  	WSA_FLAG_OVERLAPPED
);
```



IOCP的创建和绑定都是使用CreateIoCompletionPort的，具体描述为:

```c++
HANDLE CreateIoCompletionPort(
		HANDLE,					// 这里表示需要被绑定的socket
  	HANDLE,					// 这里表示iocp句柄，即socket需要绑定到的iocp
  	ULONG_PTR,			// args
  	WORD						// work 数量，传0表示为CPU*2
);
```

这里的args就有点类似起线程中的传递参数，具体用途将在`GetQueuedCompletionStatus`中用到。





 那创建和绑定操作就为:

```C++
// 传入一个无效的socket，以及一个nullptr表示创建新的iocp句柄
HANDLE iocp = CreateIoCompletionPort(INVALID_SOCKET, nullptr, 0, 0);
char* args = char[1024];
// 将socket绑定到iocp上去，接下来我们就可以使用WSDRecv来进行操作了
CreateIoCompletionPort((HANDLE)socket, iocp, (ULONG_PTR)args, 0);
```

当我们绑定好之后，如果网络IO事件完成了，可以通过`GetQueuedCompletionStatus`来获取结果。

具体描述:

```c++
// success return true, false for fail or timeout
BOOL GetQueuedCompletionStatus(
		HANDLE CompletionPort, 			// IOCP句柄
  	LPDWORD lpNumberOfBytes,		// 获取到的字节数
  	PULONG_PTR lpCompletionKey, // 这个，就是开始时传入的args
  	LPOVERLAPPED* lpOverlapped, // 重叠结构？
  	DWORD	dwMilliseconds				// 超时时间
);
```





将文件描述符给IOCP管理后，我们就要通过另外的函数获取结果。

```C++
GetQueuedCompletionStatus(CompletionPort,&BytesTransferred, (LPDWORD)&PerHandleData,(LPOVERLAPPED*)&PerIoData, INFINITE);
```

第一个参数就是IOCP句柄，第二个参数则表示这次成功的IO操作传输了多少字节，第三个则是我们之前传入的文件描述符相关数据，第四个为IO操作结果。