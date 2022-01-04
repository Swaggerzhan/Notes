# ProtoBuf基本语法以及简单RPC实现

### 1. 基本语法

protobuf一般将内容定义在`.proto`的文件中，开头由`syntax='proto2'`或者`syntax='proto3'`来定义版本，之后就可以开始定义属于自己的类型了。

protobuf中自带的类型有很多，比如比较常见的:

```protobuf
bool
int32
float
double
string
```

当然你也可以定义属于自己的自定义字段类型，其中通过`前缀关键字`来确定此字段的需求性。

```protobuf
message MyStruct{
	required int32 id = 1;
	optional string name = 2;
	repeated string hobby = 3;
}
```

`required`关键字表示此字段是 __必须的__ ，如果没有给定，那么在初始化时将报错。

`optional`关键字则表示字段是 __可有可无的__ ，如果没有给定，则初始化时为默认值。

`repeated`关键字表示此字段是 __可重复的__ ，类似动态数组。

__至于最后的数字，可以看作是一种编译成二进制后的索引`tag`，目前只需要记住如果修改索引需要重新编译，后续解释__ 。

<font color=F####>注: 在proto3语法中，已经不支持`required`了。</font>

例如:

```protobuf
syntax = 'proto2';
message Person{
	required int32 id = 1;
	required string name = 2;
	optional bool sex = 3; // true for male, false for female
}

message Class_1{
    required string class_name = 1; // class name
    repeated Person member = 2; 		// class member
}
```

以上的示例中，`Person`表示个人，`Class_1`则表示1班，其中`repeated Person member`就表示为班级中的人数。

之后我们使用`protoc`命令生成文件。

```shell
protoc person.proto cpp_out=./ # 通过person.proto生成cpp可用的文件
```

生成的C++的接口文件将有`*.pb.h`和`*.pb.cpp`文件。通过直接引用这些文件就可以操作相关的proto结构了。

通过以上的proto语法，将生成2个类，分别为`Person`和`Class_1`，这两个类就是本体了，里面封装了各种API供使用。

```c++
// 通过set_ + 字段名，可以设定一个字段的内容
Person p; // 实例化Person类
p.set_id(1);
p.set_name("test");
p.set_sex(true);
// 通过SerializeAsString可以转为10
```

如果添加的不是POD类型，而是自定义的类型，则:

```c++
// 通过 add_ + 字段名，可以添加一个自定义的字段并且返回指针
Class_1 c;
Person* p = c.add_member(); // 添加member字段
p->set_id(1);
p->set_name("test");
p->sex(true);
```

总结一下使用的API中，关于`required`和`optional`的可以大致归类为这几种，对于不同的字段操作只需要修改` _ + 字段名`即可。

```c++
// 对于字段 id 的操作(required和optional)
inline bool has_id() const;					 // 是否设置了，如果没有设置，返回false
inline void clear_id();							 // 清除字段
inline int32_t id() const;					 // 获得id值
inline void set_id(int32_t value);   // 设置id
```

对于`repeated`就比较不同了，这是一个数组类型的东西，其API:

```c++
//  Class_1 的 member
inline int member_size() const; 		// 获取数组的长度
inline void clear_member();					// 清空数组
inline Person* add_member(); 				// 添加成员，并返回成员指针
```

以上是关于生成protobuf中传输的数据的结构体。

### 2. 为RPC预留的接口

`protobuf`是为`RPC`而生的，其中预留了`RPC`的接口，通过简单的proto生成的代码来分析:

```protobuf
option cc_generic_services = true; // 生成service

message EchoRequest{
	required string data = 1;
}
message EchoRespond{
	required string data = 1;
}
service EchoServer{
	rpc Echo (EchoRequest) returns (EchoRespond);
}
```

`rpc`关键字将生成一个`Echo`的方法，参数为`EchoRequest`，返回值`EchoRespond`。以上代码通过生成，将得到:

```c++
class EchoServer_Stub;
class EchoServer : public ::PROTOBUF_NAMESPACE_ID::Service {
 protected:
  // This class should be treated as an abstract interface.
  inline EchoServer() {};
 public:
  virtual ~EchoServer();

  typedef EchoServer_Stub Stub;

  static const ::PROTOBUF_NAMESPACE_ID::ServiceDescriptor* descriptor();

  virtual void Echo(::PROTOBUF_NAMESPACE_ID::RpcController* controller,
                       const ::EchoRequest* request,
                       ::EchoRespond* response,
                       ::google::protobuf::Closure* done);

  // implements Service ----------------------------------------------

  const ::PROTOBUF_NAMESPACE_ID::ServiceDescriptor* GetDescriptor();
  void CallMethod(const ::PROTOBUF_NAMESPACE_ID::MethodDescriptor* method,
                  ::PROTOBUF_NAMESPACE_ID::RpcController* controller,
                  const ::PROTOBUF_NAMESPACE_ID::Message* request,
                  ::PROTOBUF_NAMESPACE_ID::Message* response,
                  ::google::protobuf::Closure* done);
  const ::PROTOBUF_NAMESPACE_ID::Message& GetRequestPrototype(
    const ::PROTOBUF_NAMESPACE_ID::MethodDescriptor* method) const;
  const ::PROTOBUF_NAMESPACE_ID::Message& GetResponsePrototype(
    const ::PROTOBUF_NAMESPACE_ID::MethodDescriptor* method) const;

 private:
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(EchoServer);
};

class EchoServer_Stub : public EchoServer {
 public:
  EchoServer_Stub(::PROTOBUF_NAMESPACE_ID::RpcChannel* channel);
  EchoServer_Stub(::PROTOBUF_NAMESPACE_ID::RpcChannel* channel,
                   ::PROTOBUF_NAMESPACE_ID::Service::ChannelOwnership ownership);
  ~EchoServer_Stub();

  inline ::PROTOBUF_NAMESPACE_ID::RpcChannel* channel() { return channel_; }

  // implements EchoServer ------------------------------------------

  void Echo(::PROTOBUF_NAMESPACE_ID::RpcController* controller,
                       const ::EchoRequest* request,
                       ::EchoRespond* response,
                       ::google::protobuf::Closure* done);
 private:
  ::PROTOBUF_NAMESPACE_ID::RpcChannel* channel_;
  bool owns_channel_;
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(EchoServer_Stub);
};
```

缩减得到这些重点代码，其中`EchoServer`是服务器端使用的类，`EchoServer_Stub`是客户端所使用的类。服务器通过继承`EchoServer`来重写`Echo`方法，客户端则通过`EchoServer_Stub::Echo`调用服务器上的`Echo`方法。

通过代码，我们可与清楚的看到`EchoServer::Echo`是一个虚方法，而`EchoServer_Stub::Echo`并不是，其代码为:

```c++
void EchoServer_Stub::Echo(::PROTOBUF_NAMESPACE_ID::RpcController* controller,
                              const ::EchoRequest* request,
                              ::EchoRespond* response,
                              ::google::protobuf::Closure* done) {
  channel_->CallMethod(descriptor()->method(0),
                       controller, request, response, done);
}
```

明显，我们可以看到，客户端是通过`RpcChannel`来和服务器进行沟通的，这个`RpcChannel`是一个纯虚类，也就是可以理解成protobuf给我们预留的接口吧。

##### 具体看一下RpcChannel

```c++
class PROTOBUF_EXPORT RpcChannel {
 public:
  inline RpcChannel() {}
  virtual ~RpcChannel();

  // Call the given method of the remote service.  The signature of this
  // procedure looks the same as Service::CallMethod(), but the requirements
  // are less strict in one important way:  the request and response objects
  // need not be of any specific class as long as their descriptors are
  // method->input_type() and method->output_type().
  virtual void CallMethod(const MethodDescriptor* method,
                          RpcController* controller, const Message* request,
                          Message* response, Closure* done) = 0;

 private:
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(RpcChannel);
};
```

这里先直接下结论，如果我们需要通过protobuf预留的接口来实现RPC，那么我们就需要实现`RpcChannel`方法，其本质就是通过`socket`来进行传输的，在`RpcChannel`中， __我们要做的就是将请求参数序列化发送到服务器端，并且反序列化服务器端回送的数据__ 。





## 源码分析

首先要看的是位于`google/protobuf/service.h`中的代码，其内容缩减后大致为:

```c++
class PROTOBUF_EXPORT Service {
 public:
  inline Service() {}
  virtual ~Service();
  enum ChannelOwnership { STUB_OWNS_CHANNEL, STUB_DOESNT_OWN_CHANNEL };

  virtual const ServiceDescriptor* GetDescriptor() = 0;
  
  virtual void CallMethod(const MethodDescriptor* method,
                          RpcController* controller, const Message* request,
                          Message* response, Closure* done) = 0;
  
  virtual const Message& GetRequestPrototype(
      const MethodDescriptor* method) const = 0;
  
  virtual const Message& GetResponsePrototype(
      const MethodDescriptor* method) const = 0;

 private:
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(Service);
};


class PROTOBUF_EXPORT RpcController {
 public:
  inline RpcController() {}
  virtual ~RpcController();
  virtual void Reset() = 0;
  virtual bool Failed() const = 0;
  virtual std::string ErrorText() const = 0;
  virtual void StartCancel() = 0;
  virtual void SetFailed(const std::string& reason) = 0;
  virtual bool IsCanceled() const = 0;
  virtual void NotifyOnCancel(Closure* callback) = 0;

 private:
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(RpcController);
};

class PROTOBUF_EXPORT RpcChannel {
 public:
  inline RpcChannel() {}
  virtual ~RpcChannel();

  virtual void CallMethod(const MethodDescriptor* method,
                          RpcController* controller, const Message* request,
                          Message* response, Closure* done) = 0;

 private:
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(RpcChannel);
};
```

其中的三个类均是重要的类，并且都是纯虚类。分别为`Service`、`RpcController`、`RpcChannel`。

#### 1. Service

Service是一个虚类没错，但不需要我们来实现，在编写`proto`代码的时候，具体可以看一开始的那个Echo示例，将生成两个类，分别对应客户端和服务器端的`EchoServer_Stub`和`EchoServer`就都继承于这个`Service`类，并且实现了其中的方法。

在`protobuf`中，存在许多的`Service`，每个`Service`中又存在着许多的方法`Method`。`Service`中提供了一些方法以供识别:

```c++
virtual const ServiceDescriptor* GetDescriptor() = 0;
virtual const Message& GetRequestPrototype(
      const MethodDescriptor* method) const = 0;
virtual const Message& GetResponsePrototype(
      const MethodDescriptor* method) const = 0;
```

我们先看`GetRequestPrototype`和`GetResponsePrototype`，其功能是通过传入的`MethodDescriptor`来 __获取RPC调用中的请求类型和响应类型__ ，类比之前的例子中，对应的就分别是`EchoRequest`和`EchoRespond`了，可以看到，通过函数调用返回的是一个`Message&`，并且通`New()`方法可以获得一个`Message*`的内存块，这个内存，就是我们可以使用的参数以及返回类型了。

在例子中，在使用的时候`Message`会被downcast成`EchoRequest`以及 `EchoRespond`。

注:  __编写RPC框架的时候，并不知道参数类型和响应类型是什么，只能通过downcast动态转换__ 。

接下来看CallMethod:

```c++
virtual void CallMethod(const MethodDescriptor* method,
                          RpcController* controller, const Message* request,
                          Message* response, Closure* done) = 0;
```

其参数分别为`MethodDescriptor* method`，当一个`Service`中存在许多方法时，就是通过`MethodDescriptor`来进行区分的。

`RpcController* controller`这是一个关于RPC调用的控制器，主要是用来判定调用是否完成。

`request和respond`之前我们就有大致讨论了，分别是请求参数和响应，会被downcast。

`Closure* done`是一个闭包，当完成对`respond`的操作时，启动这个回调。

当我们通过`proto`文件来生成`*.ph.h/*.ph.cc`时，`CallMethod`就被确定了，我们要做的，是重写其中的方法，上述例子中，我们就需要实现`Echo`这个方法。

#### 2. RpcController

mark



#### 3. RpcChannel

RpcChannel也是一个纯虚类，需要我们来继承重写`CallMethod`方法

```c++
virtual void CallMethod(const MethodDescriptor* method,
                          RpcController* controller, const Message* request,
                          Message* response, Closure* done) = 0;
```

具体的参数大致都有了解过了，就不再解释了，这个方法在`RpcChannel`中主要是用来进行传输的，将`request`序列化并发送到服务器，反序列化服务器的响应`response`供上层使用。



#### 4. Descriptor

`Protobuf`中，我们关注2个比较重要的`Descriptor`，分别是`ServiceDescriptor`和`MethodDescriptor`，功能如其名。

`ServiceDescriptor`的源码中，我只列出了我比较常用的:

```c++
class PROTOBUF_EXPORT ServiceDescriptor : private internal::SymbolBase {
 public:

  // The name of the service, not including its containing scope.
  const std::string& name() const;

  // The number of methods this service defines.
  int method_count() const;
  // Gets a MethodDescriptor by index, where 0 <= index < method_count().
  // These are returned in the order they were defined in the .proto file.
  const MethodDescriptor* method(int index) const;

  // Look up a MethodDescriptor by name.
  const MethodDescriptor* FindMethodByName(ConstStringParam name) const;
};
```

`name()`获取当前`Service`的名字。

`method_count()`，当前`Service`中存在的方法数量。

`method()`和`FindMethodByName()`则分别通过索引和名字来获取`MethodDescriptor`。

可以看出，`ServiceDescriptor`是一个用来描述`Service`的。

`MethodDescriptor`也一样，只列出我比较常用的:

```c++
class PROTOBUF_EXPORT MethodDescriptor : private internal::SymbolBase {
 public:
  typedef MethodDescriptorProto Proto;

  // Name of this method, not including containing scope.
  const std::string& name() const;
  // Index within the service's Descriptor.
  int index() const;
  
  // Gets the service to which this method belongs.  Never nullptr.
  const ServiceDescriptor* service() const;
};
```

`name()`当前方法名。

`index()`当前方法在`Service`中的索引排名。

`service()`当前方法所属`Service`。

<font color=F#####>具体的使用场景: 多用于获取参数类型和响应类型，例如: </font>

```c++
Message* request = service->GetRequestPrototype(methodDescriptor).New();
Message* respond = service->GetRespondPrototype(methodDescriptor).New();
```

## 简单的RPC

通过Linux系统的简单socket套接字实现一个RPC。

### 设计

RPC中可以存在多个`Service`，每个`Service`中又存有多个`Method`，我们通过`Meta.proto`中的数据来区分，并将其放到真正的`payload`前。

```protobuf
syntax = "proto2";
message Meta{
	required string service_name = 1; // Service名字
	required string method_name = 2;	// Method 名字
	required int32 payload_len = 3;		// 有效载荷长度，也就是RPC参数序列化后的长度
}
```

接下里就可以编写客户端中比较容易写的RpcChannel类

```c++
class MyChannel : public RpcChannel{
public:
  MyChannel(std::string addr, int port)
  : port_(port)
  , addr_(std::move(addr))
  {}
  virtual void CallMethod(const ::google::protobuf::MethodDescriptor* method,
                          ::google::protobuf::RpcController* controller,
                          const ::google::protobuf::Message* request,
                          ::google::protobuf::Message* respond,
                          ::google::protobuf::Closure* done
                          ) override {
    // 将request中的数据序列化
    std::string serialize_data = request->SerializeAsString();
    RpcMeta rpcMeta; // RPC头部信息
    rpcMeta.set_service_name(method->service()->name());
    rpcMeta.set_method_name(method->name());
    rpcMeta.set_data_size(serialize_data.size());

    std::string header = rpcMeta.SerializeAsString();
    int header_size = header.size();
    header.insert(0, std::string((const char*)&header_size), sizeof(int));
    std::string data = header + serialize_data;
    send(fd_, data.c_str(), data.size(), 0);
    // 接收服务器端发送来的消息
    //char len[sizeof(int)];
    //recv(fd_, len, sizeof(int), 0);
    //header_size = *(int*)len;  // header size

    char buf[2048];
    recv(fd_, buf, 2048, 0);
    respond->ParseFromString(std::string(buf));
  }
  
};
```



















