---
title: python 脚本之 subprocess 模块
date: 2019-05-18 17:53:20
tags:
 - python
 - linux
 - ACL
---

## 前言
我们的集群有很多用户组，也有很多共享存储，现在各组需要互访存储中的数据，所以需要对组目录设置 ACL 权限，组和存储较多，所以现在写一个脚本好更快的实现这一需求。

这个脚本主要是用到了`python`的`subprocess`模块执行shell 命令，并且通过`subprocess`的`stderr`得到输出时的错误信息。



## 脚本

废话不多说，直接看脚本
```python
#!/usr/bin/python2.7
# coding=utf-8

import sys
import subprocess

def getdir(group1):
  group1_dirlist.append('/ps/'+group1)
  group1_dirlist.append('/ps2/'+group1)
  group1_dirlist.append('/ps3/'+group1)
  group1_dirlist.append('/work/'+group1)
  group1_dirlist.append('/bfs1/'+group1)
  group1_dirlist.append('/bfs2/'+group1)
  group1_dirlist.append('/lustre1/'+group1)
  group1_dirlist.append('/lustre2/'+group1)


def setacl(group1_dirlist):
  cmd_list = []
  for i in group1_dirlist:
    for y in group2_list:
      cmd_list.append("/usr/bin/setfacl  -m g:"+y+":rx  "+i)
#      cmd_list.append("/usr/bin/setfacl  -b "+i)
  
  
  print "即将执行的命令："
  for cmd in cmd_list:
    print cmd 
    sub = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, shell=True)
    tmp = sub.stderr.read().split(' ')
    if 'such' in tmp:
      errdir.append(tmp[1])
    

print "被访问组：",sys.argv[1]
group1 = sys.argv[1]
print "可访问组：",sys.argv[2:]
group2_list = sys.argv[2:]

## 获取group1  应该被设置目录
group1_dirlist = []
getdir(sys.argv[1])
print sys.argv[1]+" 所有的存储目录："
print group1_dirlist


## 为group1_dirlist 设置权限

errdir = []
setacl(group1_dirlist)
errdir = list(set(errdir))
print "不存在目录"
for i in errdir:
  print i

```
