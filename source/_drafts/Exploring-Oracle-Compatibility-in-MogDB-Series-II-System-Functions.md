---
title: Exploring Oracle Compatibility in MogDB (Series II) - System Functions
date: 2024-06-07 16:35:49
tags: [MogDB, Oracle-Compatibility]
---

## 关于MogDB的兼容性设置

上一篇文章：[Exploring Oracle Compatibility in MogDB (Series I) - Data Dictionary](https://www.kamusis.me/2024/06/06/Exploring-Oracle-Compatibility-in-MogDB-Series-I-Data-Dictionary/)描述了用`ptk demo`命令快速安装MogDB，并在自动创建的`oracle_compatibility`库里体验Oracle兼容性视图。

本篇文章继续体验Oracle兼容性函数，不过，先对MogDB的几种兼容性设置进行说明。

参考MogDB文档：[CREATE DATABASE](https://docs.mogdb.io/en/mogdb/v5.0/CREATE-DATABASE#create-database)语法中的说明，**DBCOMPATIBILITY** 参数用于在创建数据库时指定兼容模式，数据库一旦创建完毕该兼容模式无法修改。

兼容模式分为以下三种：

1. DBCOMPATIBILITY=A：Oracle兼容模式

在该兼容模式下，

2. DBCOMPATIBILITY=B：MySQL兼容模式

在该兼容模式下，

3. DBCOMPATIBILITY=PG：PostgreSQL兼容模式

在该兼容模式下，



## 关于whale插件

whale插件专门用于在A兼容模式下的MogDB，为数据库提供各种Oracle兼容能力。whale插件以数据库扩展的方式体现，主要源码位于：[openGauss/Plugin](https://gitee.com/opengauss/Plugin/tree/master/contrib/whale)仓库中。



## 安装whale插件

使用`ptk demo`命令创建的测试库，会自动安装whale插件，如果是使用`ptk install`命令安装MogDB数据库，那么还需要手工另行安装whale插件。



## 体验兼容性系统函数



