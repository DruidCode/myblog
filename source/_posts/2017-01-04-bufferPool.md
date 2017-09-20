---
title: The innoDB buffer pool
date: 2017-01-04 20:00:06
tags: mysql
categories: 数据库
---

* 最近和DBA打交道，提到了InnoDB的buffer pool，所以想了解下这个概念，最好的方式当然是翻译官方文档。
* 文档版本是5.7.
* 官网地址：https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool.html

<!--more-->

#### The InnoDB Buffer Pool

InnoDB维持了一个存储空间叫buffer pool，在内存里缓存数据和索引。知道InnoDB的buffer pool如何工作，并且经常利用它访问内存中的数据是一个很重要的mysql访问调优方式。

你可以配置各种各样的InnoDB buffer pool方式来提升性能。

* 最理想的，你设置的buffer pool大小和实际的值一样，留下足够的内存给服务器上的其他程序运行，从而没有超出页。buffer pool越大，InnoDB变的越像内存数据库，只从磁盘读取一次数据，然后在之后的读取中从内存中访问数据。查看[Section 15.6.3.2, “Configuring InnoDB Buffer Pool Size”](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool-resize.html)

* 在64位系统大内存中，你可以切分buffer pool为多个部分，在并发操作中降低对内存结构的竞争。更多细节，查看[Section 15.6.3.3, “Configuring Multiple Buffer Pool Instances”](https://dev.mysql.com/doc/refman/5.7/en/innodb-multiple-buffer-pools.html).

* 你可以经常访问内存中数据，尽管有突然的操作高峰比如备份或者报告。更多细节，查看[Section 15.6.3.4, “Making the Buffer Pool Scan Resistant”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-midpoint_insertion.html).

* 你可以控制InnoDB何时和如何提供预读请求，异步预取很快会被用到的页面进buffer pool。更多细节，查看[Section 15.6.3.5, “Configuring InnoDB Buffer Pool Prefetching (Read-Ahead)”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-read_ahead.html).

* 你可以控制当后台刷新脏页面时，InnoDB是否动态调整基于工作量的刷新比率。更多细节，查看[Section 15.6.3.6, “Configuring InnoDB Buffer Pool Flushing"](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-adaptive_flushing.html).

* 你可以微调InnoDB buffer pool刷新行为部分来提高性能。更多细节，查看[Section 15.6.3.7, “Fine-tuning InnoDB Buffer Pool Flushing"](https://dev.mysql.com/doc/refman/5.7/en/innodb-lru-background-flushing.html).

* 你可以配置InnoDB在服务重启后，如何保护当前buffer pool状态，避免冗长的预热期。你也可以保存当前的buffer pool状态，在服务运行的同时。更多细节，查看[Section 15.6.3.8, “Saving and Restoring the Buffer Pool State"](https://dev.mysql.com/doc/refman/5.7/en/innodb-preload-buffer-pool.html).

#### InnoDB Buffer Pool LRU 算法

InnoDB管理一个buffer pool列表，用一个最少最近使用(LRU)算法的变体。当房间需要增加一个新页面进pool时，InnoDB会清除最少最近用的页面，增加新页面到列表中间。这个“中点插入策略”按照以下两个子列表对待列表：

* 在头部，一个新或者young页面子列表是最近访问的
* 在尾部，一个旧页面子列表是最少最近访问的

这个算法使得在新子列表中页面能被大量查询到。旧的子列表包含最少使用页面；这些页面是清除的候选。

LRU算法默认按照以下几点操作：

* 3/8的buffer pool专门给旧的子列表用
* 列表中点是新子列表尾部遇到旧子列表头部的边界
* 当InnoDB读取一个页面进buffer pool时，起初会被插入到中点（旧子列表的头部）。这个页面能被读取是因为它包含在用户的特殊操作中，比如一个SQL查询，或者是在InnoDB的自动预读取操作部分。
* 随着数据库的操作，在buffer pool中未被访问的页面通过向列表尾部移动而“老化”。新旧子列表中的页面随着其他页面变新而变旧。旧子列表中页面也随着页面被插入到中点而变旧。最终，很长时间没被使用的页面到达旧列表尾部而被清除。

默认，被查询的页面会立即移动到新子列表，意味着它们将会在很长一段时间内留在buffer pool。一个表扫描（例如一个mysqldump操作，或者没有where的select语句）会带很大的数据进入buffer poll，然后清除大量的旧数据，即使这个新数据从来不会被用。类似的，后台进程预读加载的页面，只有被移到新列表的头部才会被访问到。这些情况会经常推送有用的页面到旧子列表，然后它们成为清除的对象。关于这些行为的更多信息，查看[Section 15.6.3.4, “Making the Buffer Pool Scan Resistant”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-midpoint_insertion.html)和[Section 15.6.3.5, “Configuring InnoDB Buffer Pool Prefetching (Read-Ahead)”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-read_ahead.html).

InnoDB标准监控输出包含buffer pool和内存部分的几个领域，这些从属于buffer pool LRU算法。更多细节查看[Section 15.6.3.9, “Monitoring the Buffer Pool Using the InnoDB Standard Monitor”](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool-monitoring.html).

#### InnoDB Buffer Pool 配置选项

几个影响InnoDB buffer pool不同部分的配置选项。

* [innodb_buffer_pool_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_size)
指定buffer pool的大小。如果buffer pool比较小，而你有充足的内存，那么把buffer pool增大点可以减少查询访问InnoDB表的磁盘I/O操作，从而提升性能。在mysql的5.7.5，innodb_buffer_pool_size设置是动态的，不用重启服务你就可以配置它。查看[Section 15.6.3.2, “Configuring InnoDB Buffer Pool Size”](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool-resize.html).

* innodb_buffer_pool_chunk_size
定义了Innodb buffer pool 改变操作的块大小。查看[Section 15.6.3.2, “Configuring InnoDB Buffer Pool Size”](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool-resize.html)

* [innodb_buffer_pool_instances](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_instances)
划分buffer pool为用户指定的几个单独部分，每一个都有自己的LRU列表和相近的数据结构，这样在并发的内存读写操作中减少竞争。这个选项只有在你设置[innodb_buffer_pool_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_size)大于1GB时才生效。你指定的总大小会被划分给所有的buffer polls。为了最高的效率，可以指定一个[innodb_buffer_pool_instances](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_instances)和[innodb_buffer_pool_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_size)的组合，这样每一个buffer pool实例都至少有1g。查看[Section 14.9.2.2, “Configuring Multiple Buffer Pool Instances”](https://dev.mysql.com/doc/refman/5.7/en/innodb-multiple-buffer-pools.html).

* [innodb_old_blocks_pct](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_old_blocks_pct)
指定InnoDB给旧子列表块用的buffer pool近似百分比。值范围为5到95。默认值为37（也就是3/8的pool）。查看[Section 15.6.3.4, “Making the Buffer Pool Scan Resistant”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-midpoint_insertion.html).

* [innodb_old_blocks_time](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_old_blocks_time)
指定一个页面插入到旧子列表后必须停留多少ms，当它第一次被访问移到新子列表前。如果值是0，那么被移到旧子列表的页面将会立即移动到新子列表，当它第一次被访问的时候，无论插入后访问发生了多久。如果值大于0，页面会停留在旧子列表知道一个访问发生，至少第一次访问后许多ms。比如，值是1000会使页面停留在旧子列表1秒在第一次访问之后，移动到新子列表之前。

设置[innodb_old_blocks_time](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_old_blocks_time)值超过0可以防止一次表扫描扫大量的新子列表，只为了这次扫描。一次页面扫描读取行被快速连续访问多次，但是之后这个页面却是没用的。如果[innodb_old_blocks_time](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_old_blocks_time)设置的值比处理页面的时间大，那么在旧子列表的和该到列表尾部的页面将被很快的清除。这样，只被一次表扫描用的页面不会对新子列表中的大量使用的页面造成损害。

[innodb_old_blocks_time](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_old_blocks_time)可以在运行时设置，所以你可以在执行诸如表扫描和下载操作的时候暂时改变它：

    SET GLOBAL innodb_old_blocks_time = 1000;
    ... perform queries that scan tables ...
    SET GLOBAL innodb_old_blocks_time = 0;
这个策略并不会执行，如果你的意图是想通过充满一个表的内容来热启动buffer pool。比如，基准测试经常在服务启动时做一个表或者索引扫描，因为当用了一段时间之后，数据通常都在buffer pool里面。在这种情况，把[innodb_old_blocks_time](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_old_blocks_time)设为0，至少等到热启动完成。查看[Section 15.6.3.4, “Making the Buffer Pool Scan Resistant”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-midpoint_insertion.html).

* [innodb_read_ahead_threshold](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_read_ahead_threshold)
控制线性预读的灵敏度，InnoDB经常用来预读取页面进buffer pool。查看[Section 15.6.3.5, “Configuring InnoDB Buffer Pool Prefetching (Read-Ahead)”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-read_ahead.html).

* [innodb_random_read_ahead](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_random_read_ahead)
为预读取页面进buffer pool启动随机预读技术。随机预读技术是基于已经存在buffer pool的也看来预言将会被用到的页面，不理会那些页面被读取的顺序。[innodb_random_read_ahead](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_random_read_ahead)默认是关闭的。查看[Section 15.6.3.5, “Configuring InnoDB Buffer Pool Prefetching (Read-Ahead)”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-read_ahead.html)。

* [innodb_adaptive_flushing](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_adaptive_flushing)
指定是否自动调整刷新[脏页面](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page)的比率，基于在工作中的buffer pool。自动调整刷新比率是想要防止I/O操作的爆发。这个设定是默认开启的。查看[Section 15.6.3.6, “Configuring InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-adaptive_flushing.html).

* [innodb_adaptive_flushing_lwm](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_adaptive_flushing_lwm)
当[adaptive flushing](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_adaptive_flushing)启用时，低水位线，相当于[redo log](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_redo_log)所能容纳的百分比。查看[Section 15.6.3.7, “Fine-tuning InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/5.7/en/innodb-lru-background-flushing.html)

* [innodb_flush_neighbors](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_flush_neighbors)
规定从buffer pool刷新页面是否也刷新其他[dirty pages](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page)，在同样[extent](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_extent)范围里。查看[Section 15.6.3.7, “Fine-tuning InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-adaptive_flushing.html).

* [innodb_flushing_avg_loops](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_flushing_avg_loops)
innoDB保留之前计算的刷新状态快照的迭代次数，控制[adaptive flushing](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_adaptive_flushing)回应改变[工作量](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_workload)的速度。查看[Section 15.6.3.7, “Fine-tuning InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/5.7/en/innodb-lru-background-flushing.html)

* [innodb_lru_scan_depth](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_lru_scan_depth)
一个影响[flush](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_flush)操作buffer pool的算法和启发式。主要关注i/o密集型工作量性能。具体来说，每一个buffer pool实例，page_cleaner线程扫描寻找[脏页面](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page)刷新有多深入buffer pool LRU的列表。查看[Section 15.6.3.7, “Fine-tuning InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/5.7/en/innodb-lru-background-flushing.html).

* [innodb_max_dirty_pages_pct](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_max_dirty_pages_pct)
InnoDB会尝试从buffer pool中[刷新](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_flush)数据，使得[脏页面](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page)的占比不回超过这个值。指定一个从0到99的整数。默认值为75.查看[Section 15.6.3.6, “Configuring InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-adaptive_flushing.html).

* [innodb_max_dirty_pages_pct_lwm](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_max_dirty_pages_pct_lwm)
预刷新的[脏页面](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page)低水位控制脏页面比例。默认值0来使预刷新行为完全无效。查看[Section 15.6.3.7, “Fine-tuning InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/5.7/en/innodb-lru-background-flushing.html).

* [innodb_buffer_pool_filename](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_filename)
指定文件名字，该文件保存[innodb_buffer_pool_dump_at_shutdown](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_at_shutdown)和[innodb_buffer_pool_dump_now](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_now)产生的表空间ID和页面ID。查看[Section 15.6.3.8, “Saving and Restoring the Buffer Pool State”](https://dev.mysql.com/doc/refman/5.7/en/innodb-preload-buffer-pool.html)

* [innodb_buffer_pool_dump_at_shutdown](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_at_shutdown)
指明当mysql宕机的时候是否记录缓存在buffer pool的页面，以在下次重启时缩短[预热](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_warm_up)过程。查看[Section 15.6.3.8, “Saving and Restoring the Buffer Pool State”](https://dev.mysql.com/doc/refman/5.7/en/innodb-preload-buffer-pool.html)

* [innodb_buffer_pool_load_at_startup](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_load_at_startup)
指定在mysql启动时，buffer pool[预热](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_warm_up)自动加载早些时候加载的页面.通常和[innodb_buffer_pool_dump_at_shutdown](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_at_shutdown)一起用。查看[Section 15.6.3.8, “Saving and Restoring the Buffer Pool State”](https://dev.mysql.com/doc/refman/5.7/en/innodb-preload-buffer-pool.html)

* [innodb_buffer_pool_dump_now](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_now)
立即记录缓存在buffer pool的页面。查看[Section 15.6.3.8, “Saving and Restoring the Buffer Pool State”](https://dev.mysql.com/doc/refman/5.7/en/innodb-preload-buffer-pool.html)

* [innodb_buffer_pool_load_now](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_load_now)
立即[预热](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_warm_up)buffer pool加载一些数据页面，不等mysql重启。在基准测试中将缓存恢复到已知状态很有用，或者是使mysql运行报表或维护查询后恢复正常工作负载。通常和[innodb_buffer_pool_dump_now](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_now)一起用。查看[Section 15.6.3.8, “Saving and Restoring the Buffer Pool State”](https://dev.mysql.com/doc/refman/5.7/en/innodb-preload-buffer-pool.html)

* [innodb_buffer_pool_dump_pct](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_pct)
指定每一个buffer pool读出下载最近用到的页面百分比。范围从1到100. 查看[Section 15.6.3.8, “Saving and Restoring the Buffer Pool State”](https://dev.mysql.com/doc/refman/5.7/en/innodb-preload-buffer-pool.html)

* [innodb_buffer_pool_load_abort](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_load_abort)
中断由[innodb_buffer_pool_load_at_startup](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_load_at_startup)或者[innodb_buffer_pool_load_now](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_load_now)触发的修复buffer pool内容的进程。查看[Section 15.6.3.8, “Saving and Restoring the Buffer Pool State”](https://dev.mysql.com/doc/refman/5.7/en/innodb-preload-buffer-pool.html)
