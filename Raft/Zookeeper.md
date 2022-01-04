# ZooKeeper

## 0x00 一致性问题

## 0x01 API

### CREATE

CREATE提供了一个创建文件的方法：

```
CREATE(PATH, DATA, FLAG)
```
分别传入的是全路径名，数据DATA，和表明znode类型的FLAG，CREATE语义是拍他的，如果CREATE返回True，则表明之前没有这个文件存在。 __可以使用这个特性实现类似分布式锁__ 。

### DELETE

DELETE提供删除文件的方法：

```
DELETE(PATH, VERSION)
```

传入全名路径和认为的版本号，每个znode都存在对应的一个版本号，如果znode更新，版本号也会随之更新，如果VERSION不同，DELETE操作将返回False。

### EXIST

EXIST提供一种检测的方法：

```
EXIST(PATH, WATCH)
```

传入全名路径，以及一个WATCH的关键字，如果WATCH关键字为False，那么这个API只是用于查看某个文件是否纯在
