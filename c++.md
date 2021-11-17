## C++

### 1. dynamic_cast和down cast的讨论

首先是多态，这个都清楚，比如:

```c++
class Base{
public:
 int x;
  virtual void show(){
    std::cout << "Base::show()" << std::endl;
  }
};
class Derived : public Base{
public:
  int y;
  virtual void show(){
    std::cout << "Deviced::show()" << std::endl;
  }
};

int main(){
  Base* p1 = new Base;
  Base* p2 = new Derived;
  p1->show(); // Base::show()
  p2->show(); // Deviced::show()
	delete p1;
  delete p2;
}
```

其通过内存中的虚表来进行调用，会调用到真正的属于自己类型的方法，但现在如果有一个需求，我们需要将`Base*`转为`Derived*`进行使用，对于以上例子，`p2`指针肯定可以通过，而`p1`就不行了，`p1`就是`Base*`类型，如果强制转为`Derived*`并且进行使用，会出现内存的错误。

这种将基类指针转为派生类的操作，我们称为`Downcast`。

具体在写程序的时候，我们如何判别这种downcast？有的donwcast是安全的，有的downcast则是错误的，这里可以使用C++中给出的`dynamic_cast`，动态转换，如果失败，则直接返回`nullptr`。

同样的代码，如果使用`dynamic_cast`，则:

```c++
int main(){
  Base* p1 = new Base;
  Base* p2 = new Derived;
  Derived* p3 = dynamic_cast<Derived*>(p1); // nullptr
  Derived* p4 = dynamic_cast<Derived*>(p2); // 成功
}
```

