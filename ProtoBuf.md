# ProtoBuf

### 基本语法

```protobuf
message Person{
    optional string name = 1;
    optional int32 id = 2;
}

message Group{
    optional string name = 1;
    repeated Person member = 2;
}

```

其中的`optional`代表可选字段，可有可无，而且后面的数字则代表`tag`，用来标识每个字段的，比如`name`字段是一个string类型，而其标识是一1。

`repeated`代表次字段是一个可重复的，也就是编程语言中的数组。

对于protobuf来说，我们可以后续新添加其字段，但是他的`tag`总是得增加的，也就是说，当`tag`最小为3时，后续新增加的字段的`tag`最小也要为4。不过你可以通过`reserved`来提前保留改字段，以便后续的使用。

```protobuf
message Person {
    optional string name = 1;
    reserved 2; // 提前保留，后续可以增加tag为2的字段
    optional int64 uid = 3;
    // 后续可以一直增加
    optional int64 newUid = 4; // 新增加的字段
}
```

### API

```C++
string SerializeAsString(); // 将对象序列化成字符串
bool ParseFromString(string); // 将序列号字符串恢复成对象

// 以上的序列化是人不可读的，下面生成的为可读对象
string DebugString(); /* human-readable text proto format */ 
bool TextFormat::Parser::ParseFromString(string, Message*) // 反序列化
```

对于一开始的那个例子

```protobuf
message Person{
    optional string name = 1;
    optional int32 id = 2;
}

message Group{
    optional string name = 1;
    repeated Person member = 2;
}
```

使用以上生成的文件

```C++
#include "group.pb.h"

int main () {
    Group group;
    group.set_name("Class 1"); // 也就是Group中tag为1的字段
    Person* p1 = group.add_member(); // tag为2的一个数组，会返回一个初始化的数组成员
    p1->set_name("Swagger"); // Person中tag为1的字段
    p1->set_id(1) // Person中tag为2的字段
    Person* p2 = group.add_member(); // tag为2
    p2->set_name("0xdeadbeef"); // Person中tag为1的字段
    p1->set_id(2) // Person中tag为2的字段
    std::string s = group.SerializeAsString(); // 序列化为字符串
}
```

### protoc

protoc编译器编译proto文件生成对应语言可用的包

```shell
protoc --proto_path=基础目录 --cpp_out=目标目录 源文件
```

例如

```shell
protoc --cpp_out=src src/person.proto
```




