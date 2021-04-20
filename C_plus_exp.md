# C++ exp

### 1. static 关键字

```C++
#include <iostream>

static void f4();
static int t;

class Test{
public:

    Test();
    void f1();
    void f2();
    static void f3();
    
public:
    int x;
    int y;
    static int z; /* static 关键字 */
};


void f5(){
    static int i = 10;
    i++;
}
```

(C++)
关于static修饰的变量，如变量z，它是个`类变量`， __所有由这个类实例出来的对象都可以访问，并且它只有一份copy，位于全局数据区域。__ 类变量必须在`类外定义`，如果没有赋予初始值，由于位于全局数据区， __所以一般为0 (动态区如栈或者堆一般为垃圾值)。__ sizeof计算一个对象的大小不包含static修饰的变量。

被static修饰的类函数也是如此，它将变成`类方法`，这个类实例出来的对象都可以访问到。只不过有一个地方相反， __即static方法里访问不到this指针！。__ 
(C)
对于全局静态变量如`t`， __保存在全局数据区，它不被其他文件所访问，如果文件`File1`中包含此全局静态变量。则`File2`文件是无法访问到的，即使是使用extern提前声明都不行。__ 
局部静态变量`i`同样保存在 __全局数据区__ ，所以将导致整个函数调用结束后， __`f5()`函数中的其他数据会被销毁(其他数据放在栈中)，下次调用`f5()`时，`i`的数据和上次调用一样。__
静态函数`f4()`函数只能在定义的文件中使用，和全局静态变量相似。

注： __当一个头文件只有头文件存在时，那么所有文件只要包含此头文件即可访问其中的static修饰的变量或者函数，如果这个头文件存在对应的.c或者.cpp文件存在，那么其他文件无法访问static修饰的变量或者函数。__ (本质是编译器对.h等头文件等的不同处理方式)

暂时理解：单独的一个.h文件被include时会被单独的复制代码到include它的文件中，编译的时候也不会出现单独.o文件，它和include它的文件融合为一体了。也就没有了链接器找不到的问题。

### 2. 类的构造函数

```C++
class Gun{
public:
    int money;
    int distance;

public:
    
    /**
     * 列表构造函数
     **/
    Gun(int m, int dis):money(m), distance(dis){
        printf("构造函数1.0\n");
    }
    
    /**
     * 赋值构造函数
     **/
    Gun(int m, int dis){
        this->money = m;
        this->distance = dis;
    }
    
    Gun(){
        this->money = 1000; /* 默认值 */
        this->distance = 800;
        printf("构造函数2.0\n");
    }
    
    /**
     *  拷贝构造函数
     */
    Gun(Gun &g){
        this->money = g.money
        this->distance = g.distance;
        printf("构造函数3.0\n");
    }
    
};
```

类在被实例化的时候都会调用 __构造方法__ ，即使没有显式的写出，编译器也会为其添加所谓的构造函数，构造函数可以使用列表方式，这种方式的效率更高一些。
 __构造函数可以有多个，即重载__ ，如上例子，使用`Gun *g = new Gun(1, 2);`调用`1.0方式的构造函数`，而`Gun *g = new Gun();`则会调用`2.0的构造函数`。
 拷贝构造函数在一个对象进行拷贝时候会调用，如果没有显式的定义，它也会存在，只不过会出现一些`浅拷贝问题`， __即拷贝后的对象和拷贝前的对象中的指针指向相同区域的内存。__
 
### 3. 右值引用

```C++
/* a可以取地址，是左值，10不可以取地址，是右值 */
int a = 10;


class A{
public:
    A();
    A(A& a);

};

A a1; // 这里没有调用拷贝构造函数
/* a2可以取地址，是左值，A()不可以取地址，是右值 */
A a2 = A(); // 这里其实会调用拷贝构造函数！
```

区分左右值的方式最直观的就是判断是否可以取地址。
```C++
int a = 10;
int &a_ref = a; // 左值引用指向左值，正确
int &b_ref = 10; // 左值引用指向右值，错误
const &c_ref = 10; // const左值引用可以指向右值，正确
```
由于const左值引用不会改变值，所以它可以指向右值，所以许多函数将 `const &`作为参数，可以降低拷贝成本。


### 4. const关键字

#### 面向过程使用时:

```C++
/* 这两者本质没啥区别 */
int const x = 10;
const int y = 20;


const int *p1 = &x; /* p1指向的数值不可修改 */
/* 这里我们不能通过*p1修改x的值，但是我们可以直接修改x的值 */
x = 20; // 这里将导致*p1的值变为20
/* 不能修改*p1的值，但是我们可以将p1赋值为其他的地址 */
p1 = &y; // 这里将导致*p1的值变为20


int const *p2 = &x; /* p2的数值不可修改 */
/* 我们可以使用x直接修改*p2的值 */
x = 20; // 这里将导致*p2的值为20
/* 我们也可以使用*p2修改 */
*p2 = 20; // 这里将导致*p2的值为20

/* 指针和指向的值都不可以改 */
const int* const p1 = &x;
x = 1000; // 即时如此我们还是可以通过修改x来修改*p1的值
```

我们先来看一段代码和它的输出结果，将让我们看到const字段的本质。

```C++
     1  #include <iostream>
     2  #define BUF 1024
     3
     4
     5  int main(){
     6
     7          const int t = 10;
     8          int const x = 20;
     9
    10          int buf_size = BUF;
    11
    12
    13          int *p = (int*)&t;
    14          *p = 99;
    15
    16          printf("t  = %d\n", t);
    17          printf("*p = %d\n", *p );
    18
    19
    20          return 0;
    21
    22
    23  }
```

__输出结果将打印10和99__ ，也就是说我们虽然修改了`t处地址的值`，但是它打印的值还是为10，这怎么可能，之前看到有人说是`预编译器`搞的鬼， __它会将其const的值替换为对应的数字__ ，但是当我使用-E字段后，并没有发生如此，只有BUF的值被替换为1024。
我们使用gdb调试发现：
```shell
(gdb) p t
$6 = 10
(gdb) p &t
$7 = (const int *) 0x7fffffffe574
(gdb) p p
$8 = (int *) 0x7fffffffe574
(gdb) p *p
$9 = 10
(gdb) n
Breakpoint 3, main () at target.cpp:20
20              return 0;
(gdb) p t
$10 = 99
(gdb) p *p
$11 = 99
```
没问题，t处地址的值确实改为99了，但为啥会打印出10？
使用汇编查看一下当时发生了什么
```shell
   0x0000000000400757 <+48>:    mov    QWORD PTR [rbp-0x10],rax
   0x000000000040075b <+52>:    mov    rax,QWORD PTR [rbp-0x10]
   0x000000000040075f <+56>:    mov    DWORD PTR [rax],0x63
=> 0x0000000000400765 <+62>:    mov    esi,0xa
   0x000000000040076a <+67>:    mov    edi,0x400885
   0x000000000040076f <+72>:    mov    eax,0x0
   0x0000000000400774 <+77>:    call   0x4005e0 <printf@plt>
   0x0000000000400779 <+82>:    mov    rax,QWORD PTR [rbp-0x10]
```
可以看到，当代码跑到`printf`的时候 __将寄存器esi的值压入0xa__ 也就是10，之后 `call printf@plt`调用了printf打印出了10，其实是编译器在编译的时候将t改为了固定的值！



#### 面向对象使用时:

```C++
class Test{

public:

    const int test; // 不赋值，只能在对象初始化的时候初始化(列表初始化)
    
    int x;
    int y;
    
    mutable int z;
    
public:

    Test(int t): test(t){}

    void test1() const; // 常成员函数
    
    void test2();

};
```
其中 __test1()常成员函数可以访问x和y的值，但是无法修改普通对象中的x和y的值__ ，这是由于const修饰的就是`this`指针。也就是`void test1() const;`其实是`void test1(const Test* this);`，还记得之前的定义么，这种定义的意思表示的是 __无法修改`this`指向的地址处的值，所以自然无法修改其中x和y的值。__ 

如果在必须在常成员函数中修改某个特定的值，则可以对其定义`mutable`关键字，表示这个值总是可以被改变的， __如其中的`z`变量是可以被常成员函数所改变的。__

```C++
const Test* t = new Test(10);
t->test1();
```
常成员函数只能调用常成员函数。
常对象只能调用常成员函数。

```C++
/* 限制的是指针指向的值，所以不能调用其他方法，只能调用常成员函数 */
const Test* t = new Test(10);
/* 限制的是指针的值，对于指针指向的值不影响，所以所有函数都能调用 */
Test* const t = new Test(20);

```

### 5. friend关键字

友元函数以及友元类
C++中类存在许多的属性如public，protected，private。
public: 类中所有方法，子类所有方法，类外方法都可以访问。
protected: 类中所有方法，子类所有方法都可以访问，类外不可访问。
private: 类中所有方法可以访问，子类和类外不可访问。

friend关键字的用处在于，当一个类中有许多封装好的private属性，但有不得已需要开放一些访问的途径，即可以将一个函数设置为`friend (友好的函数)`，那么函数中即可访问此类所有的属性。

```C++
class Student{

private:
    int money;
    
public:
    Student(int m): money(m){}

    ~Student(){}
    
    friend void cat_money(Student &stu); // 声明了一个友元函数
};

void cat_money(Student &stu){
    cout << stu->money << endl; // 访问了其私有属性
}
```

以上方法就可定义一个友元函数，除此之外，还能定义一个友元类，当一个类被定义为友元类，那么类中的所有方法都能访问到其私有属性等。

```C++
class Parent;

class Student{
private:
    int money;
    
public:
    Student(int m): money(m){}
    ~Student(){}
    
    friend class Parent; // 将parentl类设置为友元类
};

class Parent{
public:
    Parent(){}
    ~Parent(){}
    
    cat_money(Student &stu){
        cout << stu->money << endl; // 访问student中的私有属性。
    }
};
```

__友元类和友元函数不可继承和传递，__ 如有一个类`A`，它将类`B`设置为 __友元类__ ，类`B`又有一个子类`C`，那么`C`无法访问`A`中的私有属性。且派生类也不会收到其影响，还是以上的例子，现在类`A`有一个子类`D`， __那么类`B`是无法访问到`D`中的私有属性的。__ 
 一般不建议这样做，将一个类设置为友元类将暴露太多封装的细节。
 
### 6. class类中的一些需要记住的知识

```C++
class A;
class B: public A;
class C: public C;
```
那么以上的构造函数是从`A()`开始，到`C()`，析构函数是`~C()`开始直到`~A()`。
类的继承还涉及到public继承，protected继承，private继承等。

```C++
基类中成员   继承方式       派生类中的成员
public       public         public
protected    public         protected
private      public        无权访问


基类中成员    继承方式       派生类中的成员
public       protected      protected
protected    protected      protected
private      protected      无权访问

基类中成员    继承方式       派生类中的成员
public       private        private
protected    private        private
private      private        无权访问
```

### 7. operator关键字

operator关键字可以将某些操作符号进行重载，比如说`==`

```C++
class Student{
public:
    int age;
    bool operator==(const Student &stu){
        return this->age == stu->age;
    }
};

int main(){
    Student t1;
    t1.age = 20;
    Sutdent t2;
    t2.age = 22;
    if (t1 == t2)
        cout << "yes" << endl;
    else
        cout << "no" << endl;
}
//输出NO
```

即将右边的t2当作函数operator的参数传入。


 





