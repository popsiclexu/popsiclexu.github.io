---
layout:     post
title:      "数据库性能测试Benchmark：TPC-H工具使用"
subtitle:   TPC-H生成不同大小数据集及导入PostgreSQL"
date:       2018-07-28 
author:     "Xu Zhenxue"
header-img: "img/post-bg-universe.jpg"
tags:
    - 数据库
    - PostgreSQL
---

> 滚动查看全文

## 前言
TPC-H 主要目的是评价特定查询的决策支持能力，强调服务器在数据挖掘、分析处理方面的能力。查询是决策支持应用的最主要应用之一，数据仓库中的复杂查询可以分成两种类型：一种是预先知道的查询，如定期的业务报表；另一种则是事先未知的查询，称为动态查询（Ad- Hoc Query）。

通俗的讲，TPC-H就是当一家数据库开发商开发了一个新的数据库操作系统，采用ＴｐＣ－Ｈ作为测试基准，来测试衡量数据库操作系统查询决策支持方面的能力

在研究生阶段的数据库课程学习中，我们主要用TPCH生成不同大小的数据集，并对TPCH提供的SQL语句在POSTGRESQL数据库下进行查询优化。在这片文章中主要描述了：针对PostgreSQL数据库，如何配置编译TPCH源码并生成不同数据量的数据集导入到数据库中

## 环境

操作系统：Linux（Ubuntu16.04）   
TPC-H工具：2.17.3  
PostgreSQL版本：9.6.0  
TPC-H工具下载网站：http://www.tpc.org/tpch/

## 生成dbgen和qgen
1. 解压TPCH-tools工具在dbgen目录下找到并更改makefile.suite 生成dbgen

```
#makefile.suite 的更改参数如下

CC      = gcc
# Current values for DATABASE are: INFORMIX, DB2, TDAT (Teradata)
#                                  SQLSERVER, SYBASE, ORACLE, VECTORWISE
# Current values for MACHINE are:  ATT, DOS, HP, IBM, ICL, MVS, 
#                                  SGI, SUN, U2200, VMS, LINUX, WIN32 
# Current values for WORKLOAD are:  TPCH

DATABASE = POSTGRESQL     #程序给定参数没有postgresql ，修改tpcd.h 添加POSTGRESQL脚本
MACHINE = LINUX
WORKLOAD = TPCH

```
2. 由于TPCH数据库参数没有PostgreSQL数据库选项，需要自己增加PG数据的脚本，在dbgen目录下更改tpcd.h文件

```
//修改tpcd.h

#ifdef POSTGRESQL
#define GEN_QUERY_PLAN  "EXPLAIN"      
#define START_TRAN      "BEGIN TRANSACTION"
#define END_TRAN        "COMMIT;"
#define SET_OUTPUT      ""
#define SET_ROWCOUNT    "LIMIT %d\n"
#define SET_DBASE       ""
#endif /* VECTORWISE */
```
3. 保存修改在终端中cd到dbgen目录下，执行下列命令 

```
// 保存更改，在dbgen目录下执行

make -f makefile.suite

// 执行成功后在dbgen目录下生成dbgen和qgen文件
```

## 运行dbgen生成.tbl数据

```
// 在dbgen目录下执行
./dbgen -s 1 -f   #-s 1 表示生成1G数据  -f覆盖之前产生的文件

// 执行成功后会在dbgen目录下生成八个.tbl文件，可通过下列命令查看（在dbgen目录下）

ls *.tbl

// 看到产生八个tbl文件
```

## 建立数据库
在postgresql中建立tpch数据库，并创建表，相关表的创建语句可以从dss.ddl中复制

```
CREATE TABLE NATION  ( N_NATIONKEY  INTEGER NOT NULL,
                            N_NAME       CHAR(25) NOT NULL,
                            N_REGIONKEY  INTEGER NOT NULL,
                            N_COMMENT    VARCHAR(152));

CREATE TABLE REGION  ( R_REGIONKEY  INTEGER NOT NULL,
                            R_NAME       CHAR(25) NOT NULL,
                            R_COMMENT    VARCHAR(152));

CREATE TABLE PART  ( P_PARTKEY     INTEGER NOT NULL,
                          P_NAME        VARCHAR(55) NOT NULL,
                          P_MFGR        CHAR(25) NOT NULL,
                          P_BRAND       CHAR(10) NOT NULL,
                          P_TYPE        VARCHAR(25) NOT NULL,
                          P_SIZE        INTEGER NOT NULL,
                          P_CONTAINER   CHAR(10) NOT NULL,
                          P_RETAILPRICE DECIMAL(15,2) NOT NULL,
                          P_COMMENT     VARCHAR(23) NOT NULL );

CREATE TABLE SUPPLIER ( S_SUPPKEY     INTEGER NOT NULL,
                             S_NAME        CHAR(25) NOT NULL,
                             S_ADDRESS     VARCHAR(40) NOT NULL,
                             S_NATIONKEY   INTEGER NOT NULL,
                             S_PHONE       CHAR(15) NOT NULL,
                             S_ACCTBAL     DECIMAL(15,2) NOT NULL,
                             S_COMMENT     VARCHAR(101) NOT NULL);

CREATE TABLE PARTSUPP ( PS_PARTKEY     INTEGER NOT NULL,
                             PS_SUPPKEY     INTEGER NOT NULL,
                             PS_AVAILQTY    INTEGER NOT NULL,
                             PS_SUPPLYCOST  DECIMAL(15,2)  NOT NULL,
                             PS_COMMENT     VARCHAR(199) NOT NULL );

CREATE TABLE CUSTOMER ( C_CUSTKEY     INTEGER NOT NULL,
                             C_NAME        VARCHAR(25) NOT NULL,
                             C_ADDRESS     VARCHAR(40) NOT NULL,
                             C_NATIONKEY   INTEGER NOT NULL,
                             C_PHONE       CHAR(15) NOT NULL,
                             C_ACCTBAL     DECIMAL(15,2)   NOT NULL,
                             C_MKTSEGMENT  CHAR(10) NOT NULL,
                             C_COMMENT     VARCHAR(117) NOT NULL);

CREATE TABLE ORDERS  ( O_ORDERKEY       INTEGER NOT NULL,
                           O_CUSTKEY        INTEGER NOT NULL,
                           O_ORDERSTATUS    CHAR(1) NOT NULL,
                           O_TOTALPRICE     DECIMAL(15,2) NOT NULL,
                           O_ORDERDATE      DATE NOT NULL,
                           O_ORDERPRIORITY  CHAR(15) NOT NULL,  
                           O_CLERK          CHAR(15) NOT NULL, 
                           O_SHIPPRIORITY   INTEGER NOT NULL,
                           O_COMMENT        VARCHAR(79) NOT NULL);

CREATE TABLE LINEITEM ( L_ORDERKEY    INTEGER NOT NULL,
                             L_PARTKEY     INTEGER NOT NULL,
                             L_SUPPKEY     INTEGER NOT NULL,
                             L_LINENUMBER  INTEGER NOT NULL,
                             L_QUANTITY    DECIMAL(15,2) NOT NULL,
                             L_EXTENDEDPRICE  DECIMAL(15,2) NOT NULL,
                             L_DISCOUNT    DECIMAL(15,2) NOT NULL,
                             L_TAX         DECIMAL(15,2) NOT NULL,
                             L_RETURNFLAG  CHAR(1) NOT NULL,
                             L_LINESTATUS  CHAR(1) NOT NULL,
                             L_SHIPDATE    DATE NOT NULL,
                             L_COMMITDATE  DATE NOT NULL,
                             L_RECEIPTDATE DATE NOT NULL,
                             L_SHIPINSTRUCT CHAR(25) NOT NULL,
                             L_SHIPMODE     CHAR(10) NOT NULL,
                             L_COMMENT      VARCHAR(44) NOT NULL);
```


## 导入数据

生成的tbl数据每一行的末尾会有一个“|”，导致PG数据库读取时报错，需要将最后一个“|”去掉，在dbgen目录下找到print.c, 注释145和147行，如下所示


```
      }

//#ifdef EOL_HANDLING
        if (sep)
//#endif /* EOL_HANDLING */
        fprintf(target, "%c", SEPARATOR);

        return(0);
}
```

最后，将数据导入PostgreSQL数据库中

```
su - postgres  //进入PostgreSQL数据库
psql  //执行sql语句
\c tpch  //切换到tpch数据库

Copy region FROM '/2.17.3/dbgen/tbl/region.tbl' WITH DELIMITER AS '|';
Copy nation FROM '/2.17.3/dbgen/tbl/nation.tbl' WITH DELIMITER AS '|';
Copy part FROM '/2.17.3/dbgen/tbl/part.tbl' WITH DELIMITER AS '|';
Copy supplier FROM '/2.17.3/dbgen/tbl/supplier.tbl' WITH DELIMITER AS '|';
Copy customer FROM '/2.17.3/dbgen/tbl/customer.tbl' WITH DELIMITER AS '|';
Copy lineitem FROM '/2.17.3/dbgen/tbl/lineitem.tbl' WITH DELIMITER AS '|';
Copy partsupp FROM '/2.17.3/dbgen/tbl/partsupp.tbl' WITH DELIMITER AS '|';
Copy orders FROM '/2.17.3/dbgen/tbl/orders.tbl' WITH DELIMITER AS '|';

```

## 给各表加约束条件

数据表的约束条件存放在dss.ri 文件中，复制并做相应更改在数据库中执行生成相关约束。
```
-- For table REGION
ALTER TABLE REGION
ADD PRIMARY KEY (R_REGIONKEY);

-- For table NATION
ALTER TABLE NATION
ADD PRIMARY KEY (N_NATIONKEY);

ALTER TABLE NATION
ADD FOREIGN KEY (N_REGIONKEY) references REGION;

COMMIT WORK;

-- For table PART
ALTER TABLE PART
ADD PRIMARY KEY (P_PARTKEY);

COMMIT WORK;

-- For table SUPPLIER
ALTER TABLE SUPPLIER
ADD PRIMARY KEY (S_SUPPKEY);

ALTER TABLE SUPPLIER
ADD FOREIGN KEY (S_NATIONKEY) references NATION;

COMMIT WORK;

-- For table PARTSUPP
ALTER TABLE PARTSUPP
ADD PRIMARY KEY (PS_PARTKEY,PS_SUPPKEY);

COMMIT WORK;

-- For table CUSTOMER
ALTER TABLE CUSTOMER
ADD PRIMARY KEY (C_CUSTKEY);

ALTER TABLE CUSTOMER
ADD FOREIGN KEY (C_NATIONKEY) references NATION;

COMMIT WORK;

-- For table LINEITEM
ALTER TABLE LINEITEM
ADD PRIMARY KEY (L_ORDERKEY,L_LINENUMBER);

COMMIT WORK;

-- For table ORDERS
ALTER TABLE ORDERS
ADD PRIMARY KEY (O_ORDERKEY);

COMMIT WORK;

-- For table PARTSUPP
ALTER TABLE PARTSUPP
ADD FOREIGN KEY (PS_SUPPKEY) references SUPPLIER;

COMMIT WORK;

ALTER TABLE PARTSUPP
ADD FOREIGN KEY (PS_PARTKEY) references PART;

COMMIT WORK;

-- For table ORDERS
ALTER TABLE ORDERS
ADD FOREIGN KEY (O_CUSTKEY) references CUSTOMER;

COMMIT WORK;

-- For table LINEITEM
ALTER TABLE LINEITEM
ADD FOREIGN KEY (L_ORDERKEY)  references ORDERS;

COMMIT WORK;

ALTER TABLE LINEITEM
ADD FOREIGN KEY (L_PARTKEY,L_SUPPKEY) references PARTSUPP;

COMMIT WORK;
```

## 生成查询语句

复制qgen 和dists.dss 到queries ,cd到queries目录下执行

```

./qgen -d 1 >d1.sql  //-d表示默认参数，1表示按照模板一生成sql语句

```
参考博客

http://www.cnblogs.com/joyeecheung/p/3599698.html


```
TRUNCATE TABLE REGION CASCADE;
TRUNCATE TABLE NATION;
TRUNCATE TABLE CUSTOMER;
TRUNCATE TABLE LINEITEM;
TRUNCATE TABLE ORDERS;
TRUNCATE TABLE PART;
TRUNCATE TABLE PARTSUPP;
TRUNCATE TABLE SUPPLIER;
```
