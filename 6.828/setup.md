


```
git clone https://pdos.csail.mit.edu/6.828/2018/jos.git lab
git clone https://github.com/mit-pdos/xv6-public.git




sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386
sudo apt-get install gcc-multilib g++-multilib
# if use gcc 4.8
sudo apt-get install gcc-4.8-multilib
```


如果make失败，可以改成：

qemu-system-x86_64


# refs
https://askubuntu.com/questions/541126/problem-compiling-jos-undefined-reference-to-udivdi3-and-umoddi3

