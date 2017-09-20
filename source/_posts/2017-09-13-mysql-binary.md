---
title: mysql字段类型binary和varbinary
date: 2017-09-13 23:09:45
tags: mysql
categories: 数据库
---

* 以前都没用过binary和varbinary类型，这边存中文用这两个类型
* mysql版本为5.7
* 官网地址：https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html

<!--more-->

binary、varbinary类型和char、varchar类似，除了它们是存储二进制数据，而不是非二进制字符串。也就是说它们包含字节而不是字符串。这意味着，它们有二进制字符集和校对规则，并且基于值中的字节数值比较和排序。

允许的最大长度和char、varchar一样，除了一个是字节长度，一个是字符长度。

binary和varbinary数据类型与char binary和varchar binary数据类型不同。对于后一种，binary属性不会把列当成二进制字符列。相反，它会使用列字符集的二进制(_bin)校对规则，列本身存储非二进制字符串而不是二进制字节串。比如，char(5) binary是char(5) CHARACTER SET latin1 COLLATE latin1_bin，假设默认的字符集为latin1。区别于binary(5)，存储的是5字节二进制字符，二进制字符集和校对规则。关于更多二进制字符和非二进制字符串有二进制校对规则的区别，查看[Section 10.1.8.5, “The binary Collation Compared to _bin Collations”](https://dev.mysql.com/doc/refman/5.7/en/charset-binary-collations.html)。

如果严格sql模式没有开启，并且你还给binary或varbinary列设置了超出列最大长度的值，则值会被截断至合适长度并且产生一个警告。对于缩减的例子，你可以通过开启严格sql模式来禁止插入值，从而产生一个错误（而不是一个警告）。可以查看[Section 5.1.8, “Server SQL Modes”](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html)。

当存储binary值时，它们会被填充值右填充到指定长度。填充值为0x00(0字节)。插入时右填充0x00，查询时不删除尾随字节。所有字节在比较时都有意义，包括order by和distinct操作。0x00字节和空格在比较时是不同的，0x00小于空格。

比如：对于binary(3)列，插入'a '时会变成'a \0'。插入'a\0'会变成'a\0\0'。查询时插入的值保持不变。

对于varbinary，插入不会填充，查询时不会去除任何字节。所有字节在比较时都有意义，包括order by和distinct操作。0x00字节和空格在比较时是不同的，0x00小于空格。

还有尾随的填充字节被删掉或者比较时忽略它们的例子，如果列有个唯一索引，插入的列值只有尾随字节数不同才会导致重复键错误。比如，如果表包含'a'，则尝试存储'a\0'将会引发一个重复键错误。

如果你打算用binary类型存储二进制数据，应该仔细考虑上面说的填充和删除特性，确保你检索的值和存储的值完全一样。下面的例子说明binary值的0x00填充是如何影响列值比较：

	mysql> CREATE TABLE t (c BINARY(3));
	Query OK, 0 rows affected (0.01 sec)
	
	mysql> INSERT INTO t SET c = 'a';
	Query OK, 1 row affected (0.01 sec)
	
	mysql> SELECT HEX(c), c = 'a', c = 'a\0\0' from t;
	+--------+---------+-------------+
	| HEX(c) | c = 'a' | c = 'a\0\0' |
	+--------+---------+-------------+
	| 610000 |       0 |           1 |
	+--------+---------+-------------+
	1 row in set (0.09 sec)

如果检索的值必须和没有填充存储的值一样，那么可能用varbinary或者blog数据类型的一种比较合适。