<font face="Monaco">

# NFS Ganesha compile

## Ubuntu deps

```shell
apt install nfs-common 
apt install bison flex doxygen uuid-dev libcap-dev libblkid-dev 
apt install gcc g++ gdb cmake libkrb5-dev
```

## CentOS deps

```shell
yum install gcc git cmake autoconf libtool
yum install bison flex doxygen openssl-devel
yum install krb5-libs krb5-devel libuuid-devel nfs-utils
```

## compile

拉仓库

```shell
git clone https://github.com/nfs-ganesha/nfs-ganesha
git checkout V2.7.6
git submodule update --init --recursive

cmake .. -DUSE_9P=OFF -DUSE_NLM=OFF -DUSE_NFSACL3=OFF -DUSE_RQUOTA=OFF -DUSE_NFS3=OFF
make
```


## Export && run

```shell
EXPORT
{
    Export_Id = 77;
    Path = /opt; # export path
    Pseudo = /;

    FSAL {
        Name = VFS;
    }
    Access_Type = RW;
    Disable_ACL = true;
    Squash = No_Root_Squash;
    Protocols = 4; # NFS4
}
EXPORT_DEFAULTS{
    Transports = UDP, TCP;
    SecType = sys; # perm
}
```

* -F 前台
* -f 配置文件
* -N 日志级别
* -L 日志输出

```shell
/usr/bin/ganesha.nfsd -F -L /dev/stdout -f ~/ganesha.conf -N NIV_EVENT
```

## ref

[nfs-ganesha 编译运行](https://www.jianshu.com/p/98e5cc7f984a)

</font>