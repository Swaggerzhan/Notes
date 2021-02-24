# linux signal

### signal的注册
* sighandler signal(int signum, sighandler_t handler);

signum是信号，handler为处理信号的指针

```C++
#include <iostream>
#include <signal.h>
#include <unistd.h>


void handler(int signum){
    if (signum == SIGIO)
        std::cout<<"SIGIO signal: "<<signum<<std::endl;
    else if (signum == SIGUSR1)
        std::cout<<"SIGUSR1 signal: "<<signum<<std::endl;
    else
        std::cout << "error!" << std::endl;
}


int main(){
    signal(SIGIO, handler);
    signal(SIGUSR1, handler);
    std::cout << SIGIO << ", " << SIGUSR1 << std::endl;
    for (;;){
        sleep(10000);
    }
}
```

### alarm()函数
```C++
#include <iostream>
#include <unistd.h>
#include <signal.h>


void handler(int sig){
    std::cout << "handler()" << std::endl;
    if (sig == SIGALRM){
        std::cout << "this is SIGALRM" << std::endl;
        alarm(5);
    }
}


int main(){
    int i = 1;
    signal(SIGALRM, handler);
    while(true){
        printf("sleep %d\n", i);
        sleep(1);
        i ++;
    }
}
```
alarm(unsigned int second)函数会在second秒后给进程发送SIGALRM信号
上述例子中可以通过kill -SIGALRM (pid)命令向程序发送SIGALRM信号> 
