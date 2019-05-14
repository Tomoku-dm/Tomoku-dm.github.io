---
title: Snapper 快照管理工具
date: 2018-6-17 23:24:42
tags: 
 - linux
 - storage
 - snapshoot
---
## 简介
Snapper是一个用来创建和维护快照的命令行工具，提供了基本的快照工具：创建、删除快照；对比快照之间的变化，以及撤销快照之间的操作。


### 关于快照 

  快照是对卷在某一点上进行拷贝，提供了一种恢复文件系统到之前状态的一种方法。关于快照的实现，有两种的方法
- **写时复制**Copy On Write (COW)
    - 即在数据第一次写入到某个存储位置时，首先将原有的内容读取出来，写到另一位置处（为快照保留的存储空间，此文中我们称为快照空间），然后再将数据写入到存储设备中。而下次针对这一位置的写操作将不再执行写时复制操作。从COW 的执行过程我们可以知道，这种实现方式在第一次写入某个存储位置时需要完成一个读操作（读原位置的数据），两个写操作（写原位置与写快照空间），如果写入频繁，那么这种方式将非常消耗IO时间。
- **重定向写快照** Redirect-on-write (ROW)
    - “ROW重定向写”与“COW复制写”是相对的概念，它可以避免两次写操作引起的性能损失。ROW把对数据卷的写请求重定向给了快照预留的存储空间，而写操作的重定向设计则把需要两次写才能完成的操作减少为一次写。我们知道COW的两次写包括：1、将旧数据写入快照卷;2、在数据卷写入新数据。而ROW只有写入新数据一步。使用ROW快照，数据卷存放的是上一个快照时间点的旧数据，新数据最终存放在预留的快照空间。

### sanpper 优点
- 可以使用Snapper来安装/升级软件，在安装/升级前后做快照，如果安装/升级失败，就可以快速的恢复系统到正常状态 
- Snapper可以帮助快速定位哪些配置文件做了改动，帮助定位错误，并快速撤销配置文件的修改。
- 自动做快照
    
### sanpper 缺点
- 只支持Btrfs 和 thinly-provisioned LVM 文件系统。
- 在创建快照时并没有能确保数据一致性的机制。
- 不能计算快照实际大小（未知）

  
## 配置文件

Snapper 需要为每一个卷创建一个配置文件，配置文件定义了快照的创建和维护规则。
Snapper 的每一个操作都是根据卷对应的配置文件进行的。

***创建配置文件*** 
-     snapper -c lvm_config create-config -f "lvm(ext4)" /lvm_mount

***配置文件模板***
-     /etc/snapper/config-templates/default 
***源文件位置***

Snapper 的快照数据存储在当前子卷根目录的 .snapshots 隐藏文件夹中   
-     /mnt/test/.snapshots

***列出所有的配置文件***
```
[root@45-rh72-ser test]# snapper list-configs   
配置     | 子卷     
---------+----------
test_con | /mnt/test
```
***配置文件参数***

- **TIMELINE_CREATE** :如果设置为yes，便会每小时创建一个快照。这是目前唯一一种可以自动创建快照的方式，因此强烈建议将其设置为 yes。

- **ALLOW_USERS 和（或）ALLOW_GROUPS**：分别为用户和（或）组授予权限。多个条目需要使用空格 分隔。例如，要为用户 thin_user 和 thin_group 授予权限，可运行：
    -     snapper -c web_data set-config "ALLOW_USERS=thin_user" "ALLOW_GROUPS=thin_group" 


***snapper 的三种快照迭代方式***




## 使用快照
### 快照种类
- **Pre Snapshot**：
修改前的文件系统快照。每一张前快照都有一个对应的post快照。
- **Post Snapshot**：
修改后的文件系统快照。每一张后快照都有一个对应的pre快照。
- **Single Snapshot**：
独立的快照。目的之一就是为了自动创建每小时快照。此为创建快照时的默认类型。

### 创建快照
```
# snapper -c config_name create -t pre -p (打印编号)
# snapper -c config_name create -t post -p
# snapper -c btrfs_config create --command "yum install net-tools"  实现三步操作（1.pre 2.yum 3.post） 在测试中很实用

# snapper -c lvm_config create -t single 

# snapper -c test_con list      查看所有的快照
类型   | # | 前期 # | 日期                               | 用户 | 清空     | 描述     | 用户数据
-------+---+--------+------------------------------------+------+----------+----------+---------
single | 0 |        |                                 | root |          | current  |         
pre    | 1 |        | 2017年11月16日 星期四 15时57分40秒 | root |          |          |         
single | 2 |        | 2017年11月16日 星期四 16时01分01秒 | root | timeline | timeline |         
post   | 3 | 1      | 2017年11月16日 星期四 16时09分47秒 | root |          | 

```
### 比对快照的差异

- 第一列 “+”号代表新增文件，“-”代表删除文件，“c”代表修改了文件，. 什么都没变，”t” 文件属性变了，例如链接之类的。与diff语法相同。
- 第二列  权限的改变    .没改  p改了
- 第三列  用户  
- 第四列  组  
- 第五列 extended attributes 
- 第六列 acl

```
[root@45-rh72-ser test]# snapper -c lvm_config status 3..4      看每一个快照之间的简单变化
+..... /lvm_mount/hello_file 


[root@45-rh72-ser test]# snapper -c test_con diff 10..11 /mnt/test/wwww 
--- /mnt/test/.snapshots/10/snapshot/wwww    1970-01-01 08:00:00.000000000 +0800
+++ /mnt/test/.snapshots/11/snapshot/wwww    2017-11-16 16:57:04.407000000 +0800
@@ -0,0 +1 @@
+asdasd

[root@45-rh72-ser configs]# snapper -c test_con xadiff 30..31
--- /mnt/test/.snapshots/30/snapshot/wwww
+++ /mnt/test/.snapshots/31/snapshot/wwww
  +:user.abc
  
```
### 恢复快照与删除快照

```
[root@45-rh72-ser test]# snapper -c config_name undochange 1..2 取消1到2的更改

# snapper -c btrfs_config delete 1 2   删除快照，应该先删除旧快照
```


## 过滤规则
一些文件主要用来保存系统信息，例如/etc/mtab，这类文件不希望被快照操作影响到，Snapper允许通过/etc/snapper/filters/*.txt 指定过滤项，并在快照操作中忽略指定文件或文件夹的变化。 

例如我们的btrfs中我们不希望快照跟踪/var、/tmp等，可以添加到filters，这样在以后创建的快照中就看到不到关于/var、/tmp的快照跟踪了。


