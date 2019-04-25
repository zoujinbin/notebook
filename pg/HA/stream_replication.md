## PostgreSQL 异步主从流复制部署

### 参考文档

https://www.jianshu.com/p/2d07339774c0  
https://blog.csdn.net/yanggd1987/article/details/51160154  
https://www.jianshu.com/p/12bc931ebba3  
http://www.postgres.cn/docs/10/app-pgrewind.html

### 简介

postgres在9.0之后引入了主从的流复制机制，所谓流复制，就是从服务器通过tcp流从主服务器中同步相应的数据。这样当主服务器数据丢失时从服务器中仍有备份。

与基于文件日志传送相比，流复制允许保持从服务器更新。从服务器连接主服务器，其产生的流WAL记录到从服务器， 而不需要等待主服务器写完WAL文件。

PostgreSQL流复制默认是异步的。在主服务器上提交事务和从服务器上变化可见之间有一个小的延迟，这个延迟远小于基于文件日志传送，通常1秒能完成。如果主服务器突然崩溃，可能会有少量数据丢失。

若使用同步流复制模式，那么master库会等待同步slave库返回事务状态后，才会完成当前事务。所以如果slave库发生故障，那么master库的事务会一直等待下去。因此同步流复制一般适用于1主多从（即1master + 1同步salve + n异步slave)。两台主机部署高可用最好还是用异步流复制模式。

**主服务器可读可写，从服务器只读。**


### 安装部署

#### 系统环境

| Hostname | IP | System | DB | Port | DataDir
| :----: | :----: | :----: | :----: | :----: | :----:
| master | 192.168.137.5 | Ubuntu 16.04 | PG 11.2 | 9984 | /home/postgres/data
| slave | 192.168.137.6 | Ubuntu 16.04 | PG 11.2 | 9984 | /home/postgres/data

**ps:** master和slave主机都事先安装好PG数据库

#### master主机配置

1. 创建replica用户用于进行主从同步复制

```
postgres=# create role replica login replication password 'replica';
CREATE ROLE
postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 replica   | Replication                                                | {}
 vcs       | Superuser, Create role, Create DB                          | {}

postgres=#
```

2. 修改pg_hba.conf

```
# 在最后一行添加以下内容
host    replication     replica         192.168.137.0/24        md5
```

3. 修改postgresql.conf

```
wal_level = hot_standby
hot_standby = on
max_wal_senders = 3
wal_keep_segments = 500
wal_log_hints = on  # pg_rewind同步数据目录的基础
```

4. 重启pg

```
pg_ctl restart
```

#### slave主机配置

1. 删除data目录

```
rm -rf /home/postgres/data
```

2. 使用pg_basebackup生成基础备份

```
pg_basebackup -D ~/data -Fp -Xs -v -P -h 192.168.137.5 -p 9984 -U replica
```

3. 配置recovery.conf

```
cp /opt/pg/share/postgresql/recovery.conf.sample /home/postgres/data/recovery.conf

standby_mode = on    # 说明该节点是从服务器
primary_conninfo = 'host=192.168.137.5 port=9984 user=replica password=replica'  # 主服务器的信息
recovery_target_timeline = 'latest'
```

4. 启动slave库

```
pg_ctl start
```

#### 查询主备

```
1. 执行pg_controldata命令

# master主机
postgres@master:~/data$ pg_controldata 
pg_control version number:            1100
Catalog version number:               201809051
Database system identifier:           6676408201307298606
Database cluster state:               in production

# slave主机
postgres@slave:~/data$ pg_controldata 
pg_control version number:            1100
Catalog version number:               201809051
Database system identifier:           6676408201307298606
Database cluster state:               in archive recovery
```

查看**Database cluster state**值，`in production`为主库，`in archive recovery`为备库

```
2. select * from pg_stat_replication;

# master主机
postgres=# select client_addr, sync_state from pg_stat_replication; 
  client_addr  | sync_state 
---------------+------------
 192.168.137.6 | async
(1 row)

postgres=#

# slave主机 
postgres=# select client_addr, sync_state from pg_stat_replication; 
 client_addr | sync_state 
-------------+------------
(0 rows)

postgres=#  
```

`有记录`则为主库，`否则`为备库

```
3. select pg_is_in_recovery();

# master主机
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery 
-------------------
 f
(1 row)

postgres=#

# slave主机
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery 
-------------------
 t
(1 row)

postgres=#
```

`f`则为主库，`t`为备库

```
4. ps -ef | grep postgres

# master主机
postgres@master:~/data$ ps -ef | grep postgres
root       1563   1543  0 Apr06 pts/0    00:00:00 su - postgres
postgres   1564   1563  0 Apr06 pts/0    00:00:00 -su
root       1593   1580  0 Apr06 pts/1    00:00:00 su - postgres
postgres   1594   1593  0 Apr06 pts/1    00:00:00 -su
postgres   3185      1  0 00:00 pts/0    00:00:00 /opt/pg/bin/postgres
postgres   3186   3185  0 00:00 ?        00:00:00 postgres: logger   
postgres   3188   3185  0 00:00 ?        00:00:00 postgres: checkpointer  
postgres   3189   3185  0 00:00 ?        00:00:00 postgres: background writer  
postgres   3190   3185  0 00:00 ?        00:00:00 postgres: walwriter  
postgres   3191   3185  0 00:00 ?        00:00:00 postgres: stats collector  
postgres   3192   3185  0 00:00 ?        00:00:00 postgres: logical replication launcher  
postgres   3199   3185  0 00:03 ?        00:00:00 postgres: walsender replica 192.168.137.6(59432) streaming 0/23000060
postgres   3211   1564  0 00:10 pts/0    00:00:00 ps -ef
postgres   3212   1564  0 00:10 pts/0    00:00:00 grep --color=auto postgres
postgres@master:~/data$

# slave主机
postgres@slave:~/data$ ps -ef | grep postgres
root       1592   1579  0 Apr06 pts/1    00:00:00 su - postgres
postgres   1593   1592  0 Apr06 pts/1    00:00:00 -su
root       3340   3326  0 Apr06 pts/3    00:00:00 su - postgres
postgres   3341   3340  0 Apr06 pts/3    00:00:00 -su
postgres   3400      1  0 00:03 pts/3    00:00:00 /opt/pg/bin/postgres
postgres   3401   3400  0 00:03 ?        00:00:00 postgres: logger   
postgres   3402   3400  0 00:03 ?        00:00:00 postgres: startup   recovering 000000050000000000000023
postgres   3403   3400  0 00:03 ?        00:00:00 postgres: checkpointer  
postgres   3404   3400  0 00:03 ?        00:00:00 postgres: background writer  
postgres   3405   3400  0 00:03 ?        00:00:00 postgres: stats collector  
postgres   3406   3400  0 00:03 ?        00:00:00 postgres: walreceiver   streaming 0/23000060
postgres   3418   3341  0 00:11 pts/3    00:00:00 ps -ef
postgres   3419   3341  0 00:11 pts/3    00:00:00 grep --color=auto postgres
postgres@slave:~/data$
```

`wal sender`则为主库，`wal receiver`为备库

#### 测试流复制

1. 主库执行如下命令
 
```
# master主机
postgres=# create database test;
CREATE DATABASE
postgres=# \c test
You are now connected to database "test" as user "vcs".
test=# create table test (id serial, name varchar(20));
CREATE TABLE
test=# insert into test values(1,'a'), (2, 'b');
INSERT 0 2
test=#
```

2. 备库执行如下命令查看结果

```
# slave主机
postgres=# \c test
You are now connected to database "test" as user "vcs".
test=# select * from test;
 id | name 
----+------
  1 | a
  2 | b
(2 rows)

test=#
```

#### 主备切换

1. 模拟主库宕机

```
# master主机
pg_ctl stop
```

2. 备库切换为主库

```
# slave主机
pg_ctl promote
```

3. 原主库切换成备库

```
主库降级成备库之前先使用pg_rewind同步下数据目录，防止直接启动由于时间线不一致导致的数据不一致问题
```

* 配置recovery.conf

```
# master主机
cp /opt/pg/share/postgresql/recovery.conf.sample /home/postgres/data/recovery.conf

standby_mode = on    # 说明该节点是从服务器
primary_conninfo = 'host=192.168.137.6 port=9984 user=replica password=replica'  # 主服务器的信息
recovery_target_timeline = 'latest'
```

* 启动pg

```
# slave主机
pg_ctl start
```