---
title: varchar和char区别
date: 2016-10-27 10:19:03
tags: mysql
categories: 数据库
---

翻译官方文档：
* https://dev.mysql.com/doc/refman/5.7/en/char.html

<!--more-->

char和varchar类型比较相似，但是在存储和取出有区别，在最大长度和是否在尾部保留空格也有区别。

char和varchar都有一个长度声明，定义了你想存储的最大字符数。比如，char(30)可以存储最多30个字符。

当创建表时，char列的长度会一直补足到你所声明的长度。长度可以从0到255。当存储char类型值时，会在右边补足空格直到指定长度。当取出char类型值时，尾部空格将会被移除，除非[PAD_CHAR_TO_FULL_LENGTH](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_pad_char_to_full_length)sql模式被启用。

varchar列中的值是可变长的字符串。长度可以指定从0到65535。一个varchar的有效最大长度取决于最大行大小(65535字节，在各个列之间取)和所设置的字符。可以查看[Section C.10.4, “Limits on Table Column Count and Row Size”](https://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html)。

相比于char，varchar会在数据值前缀增加1字节或者2字节长度。这个前缀的长度定义了值的字节数。如果值没有超过255字节，那么用1个字节存储长度；超过了则用2字节。

如果没有启用严格SQL模式，而你赋值给一个char或者varchar列超过了列的最大长度，那么这个值会被截取，然后生成一个警告。启用严格SQL模式后，然后产生一个错误，并且值不会被插入。查看[Section 6.1.8, “Server SQL Modes”](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html)。

对于varchar类型列，超过列长的尾部空格会在插入前删除，然后产生一个警告，和当前所使用的SQL模式无关。对于char类型列，超过的空格会被默默的从插入的值中截取，也和当前所使用的SQL模式无关。

当存储varchar时，值不会被补充。尾部空格会被保留当存储和取出时，这符合标准SQL。

以下图表说明了char和varchar存储字符串的区别，char(4)和varchar(4)列（假设列用单字节字符串存储，比如latin1）。

Value | CHAR(4) | Storage Required | VARCHAR(4) | Storage Required
------------ | ------------- | ------------ | ------------ | ------------
'' | '    ' | 4 bytes | '' | 1 byte
'ab' | 'ab  ' | 4 bytes | 'ab' | 3 bytes
'abcd' | 'abcd' | 4 bytes | 'abcd' | 5 bytes
'abcdefgh' | 'abcd' | 4 bytes | 'abcd' | 5 bytes

上面表中最后一行，只有在严格SQL模式下才会出现。如果是非严格SQL模式，超过长度的值不会被存储，并且报错。

对于 InnoDB的 COMPACT, DYNAMIC 和COMPRESSED 行格式，如果列值长度大于等于768字节，char被当作可变长度对待，当是utf8mb4时，如果最大字节长度设置为超过3也会发生。例如，当被当作可变长度类型时，一个char列的值可能被选为off-page存储。查看更多信息，[Section 15.11, “InnoDB Row Storage and Row Formats”](https://dev.mysql.com/doc/refman/5.7/en/innodb-row-format.html)。

如果值存入char(4)和varchar(4)列，从列中取出不会总是一样的，因为尾部空格会从char列中移除，并且无法复原。以下例子说明了这个差异：

    mysql> CREATE TABLE vc (v VARCHAR(4), c CHAR(4));
    Query OK, 0 rows affected (0.01 sec)

    mysql> INSERT INTO vc VALUES ('ab  ', 'ab  ');
    Query OK, 1 row affected (0.00 sec)

    mysql> SELECT CONCAT('(', v, ')'), CONCAT('(', c, ')') FROM vc;
    +---------------------+---------------------+
    | CONCAT('(', v, ')') | CONCAT('(', c, ')') |
    +---------------------+---------------------+
    | (ab  )              | (ab)                |
    +---------------------+---------------------+
    1 row in set (0.06 sec)
char和varchar列的值会被排序和比较根据赋值给这个值的字符校对。

所有的mysql校对都是PADSPACE类型。这意味着所有的char,varchar和text值在被比较的时候，不会去考虑尾部空格。在这里说的比较不包含like模式匹配操作，like中尾部空格是很重要的。例如：

    mysql> CREATE TABLE names (myname CHAR(10));
    Query OK, 0 rows affected (0.03 sec)

    mysql> INSERT INTO names VALUES ('Monty');
    Query OK, 1 row affected (0.00 sec)

    mysql> SELECT myname = 'Monty', myname = 'Monty  ' FROM names;
    +------------------+--------------------+
    | myname = 'Monty' | myname = 'Monty  ' |
    +------------------+--------------------+
    |                1 |                  1 |
    +------------------+--------------------+
    1 row in set (0.00 sec)

    mysql> SELECT myname LIKE 'Monty', myname LIKE 'Monty  ' FROM names;
    +---------------------+-----------------------+
    | myname LIKE 'Monty' | myname LIKE 'Monty  ' |
    +---------------------+-----------------------+
    |                   1 |                     0 |
    +---------------------+-----------------------+
    1 row in set (0.00 sec)
这对所有的mysql版本都一样，也不会对SQL模式所影响。

#### Note
更多MYSQL字符和校对的信息，查看[Section 11.1, “Character Set Support”](https://dev.mysql.com/doc/refman/5.7/en/charset.html)。关于存储需要的额外信息，查看[Section 12.8, “Data Type Storage Requirements”](https://dev.mysql.com/doc/refman/5.7/en/storage-requirements.html)。
    
那些尾部补足字符除去或者忽略比较的情况，如果一个列有一个唯一索引，只有在尾部有补足字符时不同，报重复键错误。比如，一个表包含'a'，当试图存储'a '会引起一个重复键错误。(**注释：唯一索引会忽略尾部空格**)

