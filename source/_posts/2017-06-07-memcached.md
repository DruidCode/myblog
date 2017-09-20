---
title: memcached的内存管理总结
date: 2017-06-19 10:14:26
tags: memcached
categories: 缓存
---

>为什么memcached（文中简写mc）的内存还有空余，而我设置的key还是会被挤掉呢？（文中memcached都指1.4.37版本）
<!-- more -->

### 安装
首先是安装，网上一大堆，需要注意的是依赖libevent

### 启动
memcached -h可以查看对应的启动参数，这里介绍几个常用的：

```
-p <num>	TCP port number to listen on (default: 11211)
		指定端口号，默认是11211
-l <addr>	interface to listen on (default: INADDR_ANY, all addresses)
         	<addr> may be specified as host:port. If you don't specify
         	a port number, the value you specified with -p or -U is
         	used. You may specify multiple addresses separated by comma
         	or by using -l multiple times
         	绑定地址，比如只允许本地访问 -l 127.0.0.1
-d		run as a daemon  守护进程运行
-u <username>	assume identity of <username> (only when run as root) 指定用户运行
-m <num>	max memory to use for items in megabytes (default: 64 MB)
		分配的内存，默认64m
-M		return error on memory exhausted (rather than removing items)
		内存满后是否返回错误或者移除item
-vv		very verbose (also print client commands/responses)
		可以看到slab分配种类
-I		Override the size of each slab page. Adjusts max item size
  		(default: 1mb, min: 1k, max: 128m) 指定slab的大小，默认1m
-n <bytes>	minimum space allocated for key+value+flags (default: 48)
		默认的key+value+flags大小
-f <factor>	chunk size growth factor (default: 1.25) chunk增长的倍率
```

所以启动可以这样
> memcached -l 127.0.0.1 -P /tmp/memcached.pid -u www -m 100 -d

### 几个名词
mc内存的几个名词：
slab：内存块，mc一次申请内存的最小单位，默认1m，可以用-I参数修改
page: 分配给Slab的内存空间
chunk: 一个slab会被划分成几个相同大小的chunk
item：我们需要保存的数据，一个chunk保存一个item，item主要存储缓存的key、value、key的长度、value的长度、缓存的时间等信息
slabclass：slab的种类，mc会根据slab划分chunk的大小不一样来分成不同种类的slab
Slab是一个内存块，它是memcached一次申请内存的最小单位。在启动memcached的时候一般会使用参数-m指定其可用内存，但是并不是在启动的那一刻所有的内存就全部分配出去了，只有在需要的时候才会去申请，而且每次申请一定是一个slab。
如图：
{% asset_img mem1.png %}
图中几个错误的地方：
在我的机器上chunk size：96*f^(n-1)
classes分了42种，item不仅仅存储了key和value

### slab分配的策略
启动时可以用-vv查看slab的分配策略，例如：
```
slab class   1: chunk size        96 perslab   10922
slab class   2: chunk size       120 perslab    8738
slab class   3: chunk size       152 perslab    6898
slab class   4: chunk size       192 perslab    5461
slab class   5: chunk size       240 perslab    4369
slab class   6: chunk size       304 perslab    3449
slab class   7: chunk size       384 perslab    2730
slab class   8: chunk size       480 perslab    2184
slab class   9: chunk size       600 perslab    1747
slab class  10: chunk size       752 perslab    1394
slab class  11: chunk size       944 perslab    1110
slab class  12: chunk size      1184 perslab     885
slab class  13: chunk size      1480 perslab     708
slab class  14: chunk size      1856 perslab     564
slab class  15: chunk size      2320 perslab     451
slab class  16: chunk size      2904 perslab     361
slab class  17: chunk size      3632 perslab     288
slab class  18: chunk size      4544 perslab     230
slab class  19: chunk size      5680 perslab     184
slab class  20: chunk size      7104 perslab     147
slab class  21: chunk size      8880 perslab     118
slab class  22: chunk size     11104 perslab      94
slab class  23: chunk size     13880 perslab      75
slab class  24: chunk size     17352 perslab      60
slab class  25: chunk size     21696 perslab      48
slab class  26: chunk size     27120 perslab      38
slab class  27: chunk size     33904 perslab      30
slab class  28: chunk size     42384 perslab      24
slab class  29: chunk size     52984 perslab      19
slab class  30: chunk size     66232 perslab      15
slab class  31: chunk size     82792 perslab      12
slab class  32: chunk size    103496 perslab      10
slab class  33: chunk size    129376 perslab       8
slab class  34: chunk size    161720 perslab       6
slab class  35: chunk size    202152 perslab       5
slab class  36: chunk size    252696 perslab       4
slab class  37: chunk size    315872 perslab       3
slab class  38: chunk size    394840 perslab       2
slab class  39: chunk size    493552 perslab       2
slab class  40: chunk size    616944 perslab       1
slab class  41: chunk size    771184 perslab       1
slab class  42: chunk size   1048576 perslab       1
```
这是启动时mc会分配的slab种类，实际上，这只是个分配策略，预定义好可能使用的slab大小，并没有真正去申请内存空间，只有当实际数据添加时，才会有slab的内存申请。当数据进来时Memcached会选择一个大于等于最接近的slab来进行存储。
比如第一次往mc添加一个10字节item时，mc会根据item大小，选择合适的slab大小如class 1，然后申请内存空间，用去一个chunk，从这可以看出，剩余的86字节确实浪费了。剩下的10921个chunk供下次有适合大小item时使用，当我们用完10922个chunk后，如果再有类似item进来，mc会再申请空间生成一个class 1的slab。
默认一个slab的大小是1m，也就是1024x1024= 1048576字节，class 1的chunk数是1048576/96= 10922，取整。

chunk的大小计算公式：

	(default_size+item_size)*f^(i-1) + CHUNK_ALIGN_BYTES

i是分类，比如slab class 1
default_size 是启动时-n参数的值，默认48字节
item_type：item结构体的长度，这个版本的是48字节
f为factor，是增长倍率，默认为1.25 -f参数指定
CHUNK_ALIGN_BYTES是修正值，为了保证chunk的大小是某个值的整数倍，在64位系统里，该值为8
slabs.c中slabs_init函数（确定chunk大小，初始化slab class的描述符）的代码：

```
/* Make sure items are always n-byte aligned */
        if (size % CHUNK_ALIGN_BYTES)
            size += CHUNK_ALIGN_BYTES - (size % CHUNK_ALIGN_BYTES);
```
            
所以，slab class 1的chunk size = 48 + 48 = 96（8的倍数）
slab class 2的chunk size = 96 x 1.25 = 120（8的倍数）
slab class 3的chunk size = 120 x 1.25 = 150（不是8的倍数）则按照上面的公式
size += 8 - （150 % 8） 得出size=152
以此类推

回到开头，为什么mc的内存还有空余，而我设置的key还是会被挤掉呢？因为大量长度不规则的key，导致新申请了很多slab，占满了内存（这里占满，并不是说实实在在的内存使用，而是有点像上自习占座一样，虽然座位上没人，但是座位已经被定了），此时，新来一个item，如果适合的slab已经满了，mc并不会将这个item塞到大一点的slab，而是另外申请一个相同大小的slab，但是分配给mc的内存已经不够了，这时候就会挤掉原来slab中的数据，如果不想挤掉原来的，可以加-M参数。
[timyang](https://timyang.net/data/memcached-lru-evictions/) 总结了几点：
> 尽量不用随机字符串作为key，使用定值，这样占用空间大小相对固定。
> 估算空间大小时候请用slab size计算，不要按value长度去计算。
> 过期的不要和不过期的数据存在一起，否则不过期的可能被踢。

### mc过期策略

mc的缓存过期机制，并不会将内存释放给系统，而是将该内存标注为可用状态。并且有以下几种策略：
1. 查询时检查过期
2. 创建item的时候检查过期
3. 执行flush命令，将所有item设置为失效
4. lru爬虫，mc默认时关闭的。一个单独的线程，清理失效的item。（redis也有这种机制）
5. LRU淘汰。先从LRU链表找有没有恰好过期的空间，有就用这个。如果没有过期空间，则分配新的空间。如果分配失败，那么往往是内存用光，则从LRU链表中把最旧的即使没过期的item淘汰掉，空间分给新的item。LRU链表插入缓存item的时候有先后顺序，所以淘汰一个item也是从尾部进行，也就是淘汰最早的item。

需要注意的一点：如果设置过期时间超过30天，实际效果为1s后过期。

参考资料：
*https://www.zybuluo.com/phper/note/443547*
*http://chenzhenianqing.cn/articles/1329.html*
*http://calixwu.com/2014/11/memcached-yuanmafenxi-neicunguanli.html*
*http://blog.csdn.net/initphp/article/details/44680115*
*https://www.dexcoder.com/selfly/article/2248*
*http://weibo.com/p/230418c019b4be0102w0tj?from=page_100505_profile&wvr=6&mod=wenzhangmod*
*https://timyang.net/data/memcached-lru-evictions/*