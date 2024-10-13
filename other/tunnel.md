# gost隧道

服务器
```shell
gost -L tls://:{数据接收端口}/:{数据转发端口}
# example
./gost -L tls://:8888/:22
```

客户端
```shell
gost -L=tcp://:{本地监听端口} -F forward+tls://{服务器地址}:{服务器接收端口}
# example
./gost -L=tcp://:8888 -F forward+tls://server.com:8888
```