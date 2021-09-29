# ucontext

在glibc中有一个头文件，定义了4个系统调用，分别是getcontext,setcontext,swapcontext,makecontext。

### getcontext

```C++
int getcontext(ucontext_t *ucp);
```

这里ucontext_t需要提前申请内存空间。getcontext主要的用途是将当前上下文保存到ucp中去。


### makecontext

```C++
void makecontext(ucontext_t *ucp, void(*func)(), int argc, ...);
```

makecontext并不是真正的make出来一个context，它需要去修改getcontext后得到的上下文，也就是说需要在getcontext调用后才可使用，并且，makecontext之前我们还需要设置一下这个`context`中的栈信息，以及后继上下文。

### setcontext

```C++
int setcontext(ucontext_t *ucp);
```

将ucp中保存好的各种上下文赋值到目前的寄存器中，如果这个函数调用成功，那么将不再返回了，进而去运行ucp上下文中的eip。

### swapcontext

```C++
int swapcontext(ucontext_t *oucp, *ucontext_t *ucp);
```

swapcontext会直接保存当前的上下文到oucp中去，然后直接将ucp中的上下文赋值到寄存器中，进而转过去运行ucp中的eip，调用成功的话将不会返回。


### 例子

```C++
void func1(){
    cout << "func1" << endl;
}

int main(){
    char* stack = (char*)malloc(1024 * 128);
    ucontext_t child,main; // 主协程上下文和子协程上下文
    getcontext(&child); // 保存当前的上下文
    child.uc_stack.ss_sp = stack; // 设置栈空间
    child.uc_stack.ss_size = 1024 * 128; // 栈空间大小
    child.uc_stack.ss_flags = 0;
    child.uc_link = &main;  // 后继上下文
    
    makecontext(&child, (void (*)(void))func1, 0); // 修改之前get到的上下文
    // 将当前上下文写到main中去，然后应用之前保存的child上下文
    swapcontext(&main, &child);
    cout << "main" << endl;
}
```

运行得到

```shell
func1
main
```

程序运行到getcontext时候将上下文保存到了child中去，之后makecontext将上下文中的eip修改为了func1()，最后使用swapcontext进行上下文交换，首先我们先将这里的上下文保存到main中，然后使child中的上下文生效，程序跳转到对应的func1中运行，打印出了func1，当func1结束后，由于uc_link被设定为了main这个上下文，程序跳转到我们之前保存的main中去，也就是swapcontext的下一行代码，进而打印出了main这个字符串。
