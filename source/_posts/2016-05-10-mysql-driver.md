---
title: php从mysql取出int数据，变成了string
date: 2016-05-10 09:41:06
tags: mysql
categories: 数据库
---

php与mysql交互
<!--more-->

参考资料：
>http://stackoverflow.com/questions/1197005/how-to-get-numeric-types-from-mysql-using-pdo#answer-1197041
>http://zhangxugg-163-com.iteye.com/blog/1894990
>http://dengxi.blog.51cto.com/4804263/1748965
>http://blog.ulf-wendel.de/2008/pdo_mysqlnd-the-new-features-of-pdo_mysql/

以前一直没注意到php从mysql取出来的数据都是string类型，无论是主键int id还是float。因为php是弱类型的语言，所以其实这也没多大关系。但是这引申出php所使用的mysql驱动等问题。

首先，php是如何与mysql交互的。PHP通过某种api（其实就是扩展），基于某种驱动或lib库与mysql server连接通信。

api有三种：mysql,mysqli和pdo。

        其中mysql扩展已经不被建议使用，它将在5.5被废弃，而在php7中被去除。

驱动有两种：libmysqlclient(MySQL client server library )和mysqlnd(MySQL native driver )。

        在5.3之前，默认使用的都是libmysql.从5.3开始mysqlnd已经内置于php源代码中，并且官方强烈建议使用这个驱动，只要在编译的时候加上就行了，比如：./configure --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-mysql=mysqlnd。
        而从5.4开始，三种api的驱动默认都将为mysqlnd，所以编译的时候不需要指定驱动了，比如：./configure --with-mysqli --with-pdo-mysql --with-mysql。

可以用下面两张图表示：
5.3之前

{% asset_img 5.3.png %}

5.3之后

{% asset_img 5.4.png %}

更多对mysqlnd的介绍，参考官方手册http://php.net/manual/zh/book.mysqlnd.php

如果使用的是旧的libmysql，那没办法，得不到mysql数据的类型，都会被转换为string。而从5.3开始使用mysqlnd驱动，就可得到，但是使用mysql扩展还是会被转换成string。
通过mysqli的MYSQLI_OPT_INT_AND_FLOAT_NATIVE参数，例如：

```php
<?php
    $mysqli = new mysqli('127.0.0.1', 'root', '', 'test');
    $query = "select * from test_int";
    $mysqli->options(MYSQLI_OPT_INT_AND_FLOAT_NATIVE, 1); 
    $result = $mysqli->query($query);
    $info = $result->fetch_array();
    var_dump($info);
```
    
而通过pdo，例如：

```php
<?php
    $pdo = new PDO('mysql:host=127.0.0.1;dbname=test', 'root', '');
    $pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
    $pdo->setAttribute(PDO::ATTR_STRINGIFY_FETCHES, false);
    foreach ($pdo->query('select * from test_int') as $row) {
    var_dump($row);
}
```
    
ATTR_EMULATE_PREPARES默认为true，需要指定；而ATTR_STRINGIFY_FETCHES默认就为false。

>需要注意的是：decimal类型的数据，即使有了以上的配置，依然还是输出为string类型。