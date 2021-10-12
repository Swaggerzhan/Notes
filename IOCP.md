# IOCP 使用

[url](https://developer.aliyun.com/article/708589)

创建IOCP的句柄。 
```C++
HANDLE CompletionPort = CreateIoCompletionPort(INVALID_HANDLE_VALUE, nullptr, 0, 0);
```

在句柄中添加对应的文件描述符，让其关联起来

```C++
CreateIoCompletionPort(sClient, CompletionPort, PerHandleData, 0);
```

其中sClient就是我们要绑定的文件描述符，第二个参数则为IOCP句柄，第三个参数是关于这个sClient的句柄数据块，0则表示根据CPU个数并发执行。

将文件描述符给IOCP管理后，我们就要通过另外的函数获取结果。

```C++
GetQueuedCompletionStatus(CompletionPort,&BytesTransferred, (LPDWORD)&PerHandleData,(LPOVERLAPPED*)&PerIoData, INFINITE);
```

第一个参数就是IOCP句柄，第二个参数则表示这次成功的IO操作传输了多少字节，第三个则是我们之前传入的文件描述符相关数据，第四个为IO操作结果。