# Linux下so和a库

在最早的时候安装brpc和braft有时候会在编译运行程序后遇到奇怪的东西，比如undefined某个symbol之类的，一般来说符号没找到这种事情最正常不过了，肯定就是库中没有这个符号，只是好奇的是为啥编译braft的时候不会发生这种事情，而编译自己的应用程序会这样？

## 依赖

这里以brpc/braft的依赖作为假设，brpc依赖ssl等库，braft又依赖brpc，我们的应用程序则依赖braft。做一个简单的对等：

basic -> brpc -> braft -> self

basic库只有一个函数：

```C++
void basic() {
    cout <<< "basic" << endl;
}
```

brpc则使用了basic：

```C++
void brpc() {
    cout << "brpc" << endl;
    basic();
}
```

同理braft：

```C++
void braft() {
    cout << "braft" << endl;
    brpc();
}
```

然后生成他们的对应动态链接库和静态库：

```shell
g++ basic.cc -c -o basic.o
ar crv libbasic.a basic.o
g++ basic.cc -shared -fPIC -o libbasic.so
```

生成brpc的静态链接库的时候，坑点就出现了：

```shell
g++ brpc.cc -c -o brpc.o
ar crv libbrpc.a brpc.o
```

这里有符号找不到，符号是basic，但是静态链接库认为你后续会给他，所以并没有报错，而当使用的时候，报错点就会在这个地方。