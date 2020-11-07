# 内存临时表

> `order by rand()` 
>
> 原理分析 - 扫描行数，查看慢查询日志
>
> ![img](https://static001.geekbang.org/resource/image/2a/fc/2abe849faa7dcad0189b61238b849ffc.png)
>
> 如果创建的表没有主键，或者把一个表的主键删掉了，那么InnoDB会自己生成一个长度为6字节的rowid来作为主键。 
>
> `rowid` ：
>
> > - 对于没有主键的InnoDB表来说，这个rowid就是主键ID
> > - 对于没有主键的InnoDB表来说，这个rowid就是由系统生成de
> > - MEMORY引擎不是索引组织表。
>
> 总结：order by rand()使用了内存临时表，内存临时表排序的时候使用了rowid排序方法。

# 磁盘临时表

> 不是所有的临时表都是内存表
>
> > tmp_table_size这个配置限制了内存临时表的大小，默认值是16M。如果临时表大小超过了tmp_table_size，那么内存临时表就会转成磁盘临时表。
>
> > 磁盘临时表使用的引擎默认是InnoDB，是由参数internal_tmp_disk_storage_engine控制的。

# 随机排序法

> ```mysql
> mysql> select max(id),min(id) into @M,@N from t ;
> set @X= floor((@M-@N+1)*rand() + @N);
> select * from t where id >= @X limit 1;
> ```
>
> ``` mysql
> mysql> select count(*) into @C from t;
> set @Y = floor(@C * rand());
> set @sql = concat("select * from t limit ", @Y, ",1");
> prepare stmt from @sql;
> execute stmt;
> DEALLOCATE prepare stmt;
> ```
>
> 
>
> 



