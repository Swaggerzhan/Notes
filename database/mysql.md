# Mysql 8.0 

## 0x00 Install

#### Download && Depends

mysql的社区版本[地址](https://dev.mysql.com/downloads/)，可以直接下载到对应的二进制版本，上传至服务器即可，ubuntu下可能需要一些依赖，有这些：

```shell
sudo apt install libaio-dev libncurses5
```

#### Prepare

然后直接解压下载包即可，这边我的mysql bin conf log data的路径是这样的：

```shell
xxxx@yyyyy:~/mysql$ ls
basedir  bin  binlog  conf  datadir  include  lib   log    tmp
```

bin和include、lib这些是解压后得到的文件，我在此基础上又创建了其他目录用来保存数据等：

* basedir
* binlog binlog存放处
* conf my.conf路径
* datadir 数据存放点
* log mysql的日志
* tmp 放mysql.sock

其中conf中的my.conf配置为：

```conf
[mysqld]
lower_case_table_names =  1
user = mysql
server_id =  1
port =  3306
default-time-zone =  '+08:00'
enforce_gtid_consistency = ON
gtid_mode = ON
binlog_checksum = none
default_authentication_plugin = mysql_native_password
datadir = ${your_mysql_root_dir}/datadir
pid-file =  ${your_mysql_root_dir}/tmp/mysqld.pid
socket =  ${your_mysql_root_dir}/tmp/mysqld.sock
tmpdir = ${your_mysql_root_dir}/tmp
skip-name-resolve = ON
open_files_limit =  65535
table_open_cache =  2000
#################innodb########################
innodb_data_home_dir =  ${your_mysql_root_dir}/datadir
innodb_data_file_path = ibdata1:512M;ibdata2:512M:autoextend
innodb_buffer_pool_size = 12000M
innodb_flush_log_at_trx_commit =  1
innodb_io_capacity =  600
innodb_lock_wait_timeout =  120
innodb_log_buffer_size = 8M
innodb_log_file_size = 200M
innodb_log_files_in_group =  3
innodb_max_dirty_pages_pct =  85
innodb_read_io_threads =  8
innodb_write_io_threads =  8
innodb_thread_concurrency =  32
innodb_file_per_table
innodb_rollback_on_timeout
innodb_undo_directory =  ${your_mysql_root_dir}/datadir
innodb_log_group_home_dir = ${your_mysql_root_dir}/datadir
###################session###########################
join_buffer_size = 8M
key_buffer_size = 256M
bulk_insert_buffer_size = 8M
max_heap_table_size = 96M
tmp_table_size = 96M
read_buffer_size = 8M
sort_buffer_size = 2M
max_allowed_packet = 64M
read_rnd_buffer_size = 32M
############log set###################
log-error =  ${your_mysql_root_dir}/log/mysqld.err
log-bin =  ${your_mysql_root_dir}/binlog/binlog
log_bin_index =  ${your_mysql_root_dir}/binlog/binlog.index
max_binlog_size = 500M
slow_query_log_file =  ${your_mysql_root_dir}/log/slow.log
slow_query_log =  1
long_query_time =  10
log_queries_not_using_indexes = ON
log_throttle_queries_not_using_indexes =  10
log_slow_admin_statements = ON
log_output = FILE,TABLE
master_info_file = ${your_mysql_root_dir}/binlog/master.info
```

#### Initialization

完成配置后，初始化mysql：

```shell
# pwd -> ${your_mysql_root_dir}
./bin/mysqld --defaults-file=./conf/my.conf --initialize-insecure --user=${your_mysql_run_user}
```

启动mysql：

```shell
./bin/mysqld --defaults-file=./conf/my.conf &
```

使用客户端连接并且创建密码，表在mysql这个数据库中：

```
./bin/mysql -S ./tmp/mysqld.sock
alter user 'root'@'%' identified by '${your_passwd}';
flush privileges;
```

这里的@'%'或者localhost是设定全部IP都可访问或者仅本地登陆。

## 0x01 LogIn

```shell
./bin/mysql -S ./tmp/mysqld.sock -uroot -p
```

TODO

## 0x02 Refs && Downloads

[CentOS7 安装 MySQL8.0（二进制）](https://learnku.com/articles/38858)
[MySQL8.0允许外部访问](https://blog.csdn.net/h996666/article/details/80921913)
[ MySQL Community Downloads](https://dev.mysql.com/downloads/)

