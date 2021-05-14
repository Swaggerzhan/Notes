# C++ 11

### 1. nullptr关键字

C语言中一般将NULL宏定义为:
```C
#define NULL ((void*)0)
```
C++中将NULL宏定义为:
```C++
#define NULL 0
```

关于0和nullptr以及NULL的区别可以用这段程序测试出来

```C++
void f(int target){
    std::cout << "f(int target)" << std::endl;
}
void f(void* target){
    std::cout << "f(void* target)" << std::endl;
}
int main(){
    f(nullptr); // f(void* target)
    f(0); // f(int target)
    f(10000); // f(int target)
    f(NULL); // 报错，存在歧义
}
```

nullptr的底层是std::nullptr_t，而且NULL在C++中一般定义为0，不过上述程序中f(NULL)确确实实报存在歧义的错误，但是如果按照定义为0的情况是应该调用f(int)的函数的。