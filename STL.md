# STL

### 1.allocator

编写一个C++所接受的allocator需要在类中包含如下7个数据类型

```C++
template<typename T>
class allocator{
public:
    typedef T           value_type; // 数据类型
    typedef T*          pointer; // 数据类型指针
    typedef const T*    const_pointer; // 数据类型const指针
    typedef T&          reference; // 数据类型引用
    typedef const T&    const_reference; // 数据类型const引用
    typedef size_t      size_type; // 类型长度，总长度！
    typedef ptrdiff_t   difference_type; // ?
};
```

除此外，其中还需要提供4个接口

```C++
template <typename T>
class allocator{
public:
    pointer allocate(); // 内存申请接口
    void    deallocate(); // 内存释放接口
    void    construct(); // 所存对象的构析函数调用接口
    void    destroy();  //所存对象的析构函数调用接口
};
```