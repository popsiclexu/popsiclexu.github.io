---
layout: post
title: Greenplum集群部署总结
subtitle: The summary of installing Greenplum Database cluster
date: 2019-03-23T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Greenplum
  - 数据库

---



> 滚动阅读全文

## 前言 
Greenplum是一种基于PostgreSQL数据库开发的大规模并行计算（MPP）数据库服务器，其架构特别针对管理大规模分析型数据仓库以及商业智能工作负载而设计。  
网上关于Greenplum集群安装部署的教程很多，数据库的编译安装可以参照gp的[Github官网教程](https://github.com/greenplum-db/gpdb), 系统的前期配置和完整教程可参考[教程](hhttps://juejin.im/post/5b7522ea6fb9a009ca7eb249?tdsourcetag=s_pctim_aiomsg)，该教程成功率较高。由于Greenplum的版本不断在更新，按照原来众多博主写的教程安装还是会踩很多坑，本文总结此次安装Greenplum所踩过的坑及解决方案，希望对大家有所帮助。

## 环境配置
OS：CentOS 7虚拟机 * 3台，1个Master节点，2个Segment节点  
Greenplum：6.0.x，编译安装

## 采坑记录

#### 一、依赖安装失败问题
编译Greenplum在执行README.CentOS.bash文件安装依赖时可能出现以下问题：
1. setuptools is too old 

```
# 升级setuptools

pip install --upgrade setuptools
```

2. 安装多个依赖报错：Cannot uninstall 'XXXXX'. It is a distutils installed project and thus we cannot accurately determine which files belong to it which would lead to only a partial uninstall.

```
# 单独安装或升级相应依赖，加上 --ignore-installed参数

pip install --upgrade   XXXXX  --ignore-installed

```

#### 二、编译安装数据库出现的问题

1. Modules/constants.h:7:18: fatal error: lber.h: No such file or directory

```
# yum安装 python-devel openldap-devel

sudo yum install python-devel
sudo yum install openldap-devel

```
2. checking Checking ORCA version… configure: error: Your ORCA version is expected to be X.X.XXX  
**每个数据库节点均要编译安装ORCA，并将配置/etc/ld.so.conf** 

```
vi /etc/ld.so.conf
# 内容如下
include ld.so.conf.d/*.conf
include /usr/local/lib
include /usr/local/lib64

vi /etc/ld.so.conf.d/usrlocallib.conf
# 内容如下
/usr/local
/usr/local/lib
/usr/local/lib64

# 刷新动态链接库缓存
ldconfig
```
3. gpcheck提示：on device (/dev/sda3) blockdev readahead value '8192' does not match expected value '16384'


```
#修改 /dev/sda3 盘的预读扇区
blockdev --setra 16384 /dev/sda3

#将修改命令写入/etc/rc.local，否则重启后会失效
echo '/sbin/blockdev --setra 16384 /dev/sda2' >> /etc/rc.local

```


4. gpcheck提示：XFS filesystem on device /dev/sda3 has 5 XFS mount options and 1 are expected

```
# 执行df查看自己的数据目录改在的磁盘，修改数据所在磁盘的参数即可，其他磁盘的报错信息可以忽略

vi /etc/fstab

# 修改前
UUID=203ac506-a2fb-4465-88ac-df2caefd3268 /data  xfs     defaults        0 0

# 修改后
UUID=203ac506-a2fb-4465-88ac-df2caefd3268 /data xfs     defaults,allocsize=16348k,inode64,noatime        0 0
```
5. 初始化时提示: /usr/local/gpdb/bin/postgres: error while loading shared libraries: libgpopt.so.3: cannot open shared object file: No such file or directory  
解决的方法也很简单，这个文件实际是在/usr/local/lib里面，把这个目录加进/usr/local/gpdb/greenplum_path.sh文件的LD_LIBRARY_PATH配置中。

## 总结
Greenplum安装主要是一些细节不注意的问题，遇到报错，认真读报错信息，缺依赖装依赖，版本过低就升级，重要的是细心就行。






