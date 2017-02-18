---
layout: post
title:  hive的并发写控制
category: data_warehouse
description: 关系型数据库有严格的事务控制，在多用户并发操作时也可保证ACID特性。而基于hive的数据仓库，不具备关系型数据库的事务特性。当一个基于hive的数据仓库有多个用户同时使用时，特别是有用户在修改schema或更改数据时，如何保证其他用户的数据一致性，达到数据隔离的效果呢？通过统一的入口访问hive，在入口处做串行化的控制，是一个有效的办法。
---

### hive的并发问题描述
关系型数据库有严格的事务控制，在多用户并发操作时也可保证ACID特性。而基于hive的数据仓库，不具备关系型数据库的事务特性。

比如A用户执行select count过程中，B用户提交insert overwrite。B用户的命令不会被阻塞，会立即执行，而A用户执行到一半的mapreduce可能就会出错。而在关系型数据库里，B的命令同样不阻塞，但由于MVCC机制，A的结果不受B影响。

又比如A用户在执行insert overwrite操作过程中，B用户提交了drop table，B的命令也会立即执行，而A的操作就很可能抛出异常。在mysql中，只有A提交了commit，B的命令才会执行。

在实际的环境中，每天都会发生业务表的DDL，这就使得业务表与数据仓库中的hive表不一致，hive表需要做DDL，甚至可能需要drop后重新做ETL。即便表schema没变，有些表会定时做全量ETL，在ETL执行过程中如果表被select，也可能产生异常。

### hive的并发问题的原因
hive天生不是面向并发写操作的，而是面向于一次写入多次读取的场景，而且hive底层依赖的mapreduce和hdfs，也没有事务或MVCC相关概念。

hive底层的文件系统依赖hdfs，做insert overwrite操作，或使用sqoop这样的ETL工具时，会先把旧文件删除，在load新文件。在这替换的过程中，如果有select在执行，相应的mapreduce操作很可能会抛出File Not Found异常。

如果hive的存储格式使用TextInputFormat，hive的schema是很弱的，类似于一个csv文件，schema只是表头，DDL只变化表头，不改变内容，这就很容易出现表头和内容错位，这又增加了DDL并发操作下，出错的可能性。

### hive并发问题的解决思路
既然hive本身不提供并发控制能力，就应该在hive之上，提供一个统一的访问入口，在入口处做处理。

* 统一访问入口：所有对hive的访问都从这个入口进入，比如为用户提供一个hive网关，又比如所有的hive操作都通过同一个调度系统提交和管理。这个约束是在保证使用方的数据准确性，是很容易推行的
* 读操作可并行，写/DDL操作做阻断：一旦有写或DDL操作提交，需要等已经在执行的，用到该表的select语句执行完，并且后续提交的select要排队等待
