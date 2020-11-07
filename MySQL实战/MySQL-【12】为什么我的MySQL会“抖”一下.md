`Innodb_io_capacity` 

# redo log的flush操作

# WAL机制 

> InnoDB在处理更新语句的时候，只做了写日志这一个磁盘操作，这个日志叫做redo log（重做日志），在更新内存写完redo log后，就返回给客户端，本次更新成功
>
> `flush` 
>
> 当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入磁盘后，内存和磁盘上的数据页的内容就一致了，称为“干净页”。
>
> 不论是脏页还是干净页，都在内存中
>
> 不难想象，平时执行很快的更新操作，其实就是在写内存和日志，而MySQL偶尔“抖”一下的那个瞬间，可能就是在刷脏页（flush）
>
> 什么情况会引发数据库的flush过程呢？
>
> * InnoDB的redo log写满了，这时候系统会停止所有更新操作，把checkpoint往前推进，redo log留出空间可以继续写。
>
>   >  这种情况要尽量避免，因为出现这种情况的时候，整个系统就不能再接受更新了，所有的更新都必须堵住。
>
> * 系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。
>
>   > 常态。
>   >
>   > InnoDB用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态：
>   >
>   > * 还没有使用的
>   > * 使用了并且是干净页
>   > * 使用了并且是脏页
>   >
>   > InnoDB的策略是尽量使用内存，因此对于一个长时间运行的库来说，未被使用的页面很少。
>   >
>   > 当要读入的数据页没在内存的时候，就必须到缓冲池中申请一个数据页。这时候只能把最久不使用的数据页从内存中淘汰掉：如果要淘汰的是一个干净页，就直接释放出来复用；但如果是脏页呢，就必须将脏页先刷到磁盘，变成干净页后才能复用。
>   >
>   > 所以，刷脏页虽然是常态，但是出现以下这两种情况，都是会明显影响性能的：
>   >
>   > * 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长
>   > * 日志写满，更新全部堵住，写性能跌为0，这种情况对敏感业务来说，是不能接受的
>   >
>   > 所以，InnoDB需要有控制脏页比例的机制，来尽量避免上面的这两种情况。
>
>   > InnoDB刷脏页的控制策略
>   >
>   > innodb_io_capacity - InnoDB所在主机的IO能力，建议把这个值设置成磁盘的IOPS。磁盘的IOPS可以通过fio这个工具来测试：
>   >
>   > ` fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest ` 
>
>   > InnoDB的刷盘速度参考因素：脏页比例，redo log写盘速度。InnoDB会根据这两个因素先单独算出两个数字。
>   >
>   > 参数innodb_max_dirty_pages_pct 是脏页比例上限，默认值是75%；
>   >
>   > F1、F2 选择最大值 按照最大值%速度来刷脏页
>   >
>   > 要尽量避免MySQL“抖”的情况，要合理地设置innodb_io_capacity的值，并且平时要多关注脏页比例，不要让它经常接近75%。
>   >
>   > 其中，脏页比例是通过`Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total` 得到的，具体的命令如：
>   >
>   > ```mysql
>   > 
>   > mysql> select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
>   > select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
>   > select @a/@b;
>   > ```
>   >
>   > 
>
> * 系统“空闲”的时候
>
> * MySQL正常关闭的时候，MySQL会把内存的脏页都flush到磁盘上，这样下次MySQL启动的时候，就可以直接从磁盘上读数据，启动速度很快。