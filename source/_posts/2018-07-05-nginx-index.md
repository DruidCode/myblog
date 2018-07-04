---
title: nginx配置location的index跳转问题
date: 2018-07-05 01:11:49
tags: nginx
categories: web server
---
好久没更新了，直到这个虚拟主机提醒我收费，才想起来。。
<!--more-->

对于 location = 的匹配，网上大部分都说是精确匹配，nginx匹配出后就会停止往后，但实际上还是有个需要注意的case。
比如下面的nginx配置：
```shell
location = / {
	index index.html
}

location /index.html {
	
}

```
假如请求/，首先会匹配到第一个location，但是由于配置了index，所以nginx会做一次内部跳转，会以/index.html再请求一次，这时就会匹配到第二个location。

可以参看官网的说明：[index](https://nginx.org/en/docs/http/ngx_http_index_module.html?&_ga=2.170492832.1041583741.1526043489-411238930.1526043489#index)。
