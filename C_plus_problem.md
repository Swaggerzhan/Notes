# C++同一个头文件中类相互引用的问题

先看一段源码
```C++
/* test.h */

class A;
class B;

class A{
public:
    B* b;
    A(B* b){
        this->b = b;
        b->RUN();
    }
};

class B{
public:
    B();
    void RUN(){
        cout<<"RUN success"<<endl;
    }
};

int main(){
    B* b = new B;
    A* a = new A(b);
}
```
运行以上代码会得到报错 __error: invalid use of incomplete type ‘class B’__
按照正常逻辑这种注入类的方式应该只没有问题的，但是问题确实出现了，解决办法时将其中的类声明和方法实现分开分别存入.h和.cpp文件就可解决。
```C++
/* test.h */
class A;
class B;

class A{
public:
    B* b;
    A(B* b);
};
class B{
    B();
    void RUN(); 
};
```

```C++
/* test.cpp */

A::A(B* b){
    this->b = b;
    b->RUN();
}

B::RUN(){
    cout << "RUN success" << endl;
}
```
具体底层原因留坑

