# gdb manual

## 0x00 easy start


## 0x01 print memory

gdb中打印某个地址的内容指令为x(examine)，格式为：`x/nfu addr`，分别解析一下其中的含义：

* x，指令examine，执行打印内存
* n，重复次数
* f，显示格式(display format)
    * x 为默认，不指定f时的输出格式，16进制
    * i 指令
    * m 内存
    * s 字符串
* u，打印的单位(unit)，默认是word
    * b byte 字节 (8bit)
    * h halfworld 双字节 (16bit)
    * w word 字 (32bit)
    * g giant 双字 (64bit)
例如：

```gdb
(gdb) x/8xw pgdir
0xf011afe8:     0x00000000      0x00000001      0x00000000      0x00000001
0xf011aff8:     0x00000000      0x00000001      0x00000000      0x00000000
```





# ref
https://sourceware.org/gdb/current/onlinedocs/gdb.html/Memory.html