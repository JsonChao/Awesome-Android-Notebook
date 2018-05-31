<!-- GFM-TOC -->
* [一、基础篇](#一基础篇)
    * [SQL基础](#sql基础)
    * [MySQL支持的数据类型](#mysql支持的数据类型)
    * [MySQL中的运算符](#mysql中的运算符)
    * [常用函数](#常用函数)
    * [图形化工具的使用](#图形化工具的使用)
   
* [二、开发篇](#二开发篇)
    * [表类型（存储引擎）的选择](#表类型（存储引擎）的选择)
    * [选择合适的数据类型](#选择合适的数据类型)
    * [字符集](#字符集)
    * [索引的设计和使用](#索引的设计和使用)
    * [视图](#视图)
    * [存储过程和函数](#存储过程和函数)
    * [触发器](#触发器)
    * [事务控制和锁定语句](#事务控制和锁定语句)
    * [SQL中的安全问题](#SQL中的安全问题)
    * [SQL Mode及相关问题](#sqlmode及相关问题)
    * [MySQL分区](#mysql分区)
* [三、优化篇](#三优化篇)
    * [SQL优化](#sql优化)
    * [优化数据库对象](#优化数据库对象)
    * [锁问题](#锁问题)
    * [优化MySQL Server](#优化mysqlserver)
    * [磁盘I/O问题](#磁盘问题)
    * [应用优化](#应用优化)
* [参考资料](#参考资料)
<!-- GFM-TOC -->

# 一、基础篇

## SQL基础

SQL（结构化查询语言）

SQL分类：

SQL主要可以划分为以下三个类别。

DDL(Data Denifition Languages)语句：数据定义语言，定义了不同的数据段、数据库、表、列、索引等数据库对象。常用的语句关键字包括create、drop、alter等。
DML(Data Manipulation language)语句：数据操作语句，用于添加、删除、更新和查询数据库记录，并检查数据完整性。常用的语句关键字主要包括insert、delete、update和select等。
DCL(Data Control Language)语句：数据控制语句，用于控制不同的数据段直接的许可和访问级别的语句。定义了数据库、表、字段、用户的访问权限和安全级别。主要的语句关键字包括grant、revoke等。

DDL语句

简单来说，就是对数据库内部的对象进行创建、删除、修改等操作的语言。
与DML的最大区别：DML只是对表内数据操作，而不涉及表的定义、结构的修改，更不会涉及其它对象。DDL语句更多地由数据库管理员（DBA）使用，开发人员很少用。

此处简单介绍相关命令~

0.连接到MySQL服务器

    mysql -uroot -p 

mysql代表客户端命令，“-u”后面跟连接的数据库用户，“-p”表示需要输入密码。

1.创建数据库

create data dbname

实例：

    create data test1
    
    use test1
    
    show tables

查看所有表

2.删除数据库

drop database dbname

实例：

    drop database test1

3.创建表

create table tablename（
column_name_1_column_type_1_constraints,
column_name_2_column_type_2_constraints,
...
column_name_n_column_type_n_constraints)

实例：

    create table emp(ename varchar(10), 
    hireadate date, 
    sal decimal(10, 2), 
    deptno int(2));

查看表的定义：

desc tablename

实例：

desc emp；

4.删除表

drop table tablenaem

实例：

    drop table emp；

5.修改表
