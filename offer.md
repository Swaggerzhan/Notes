# 剑指Offer笔记


##### sizeof 一个空类型

```C++
struct test{};
struct test2{
    test2(){}
    ~test2(){}
};
class Test1{};
class Test2{
public:
    Test2(){}
    ~Test2(){}
};
class Test3{
public:
    Test3(){}
    virtual ~Test3(){}
};
cout << sizeof(struct test) << endl; // 1
cout << sizeof(struct test2) << endl; // 1
cout << sizeof(Test1) << endl; // 1
cout << sizeof(Test2) << endl; // 1
cout << sizeof(Test3) << endl; // 8
```
空类型中需要1个字节来进行占位，virtual函数需要一个虚表指针，在32bit上为4byte，在64bit上为8byte。


##### 拷贝构造函数

```C++
class A{
public:
    A(){}
    ~A(){}
    A(A a){ // 编译报错，必须是引用
        this->x_ = a.x_; 
    }
    A(const A& a){ // 正常 
        this->x_ = a.x_; 
    }
private:
    int x_;
};
```

原因: 如果不是传引用，那么会在传递参数时进行复制，进而再次调用拷贝构造函数，进入到死循环当中。