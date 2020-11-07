## count(*)的实现方式

> 不同的MySQL引擎中，count(*)有不同的实现方式
>
> - MyISAM引擎把一个表的总行数存在了磁盘上，因此执行count(*)的时候会直接返回这个数，效率很高
> - InnoDB比较麻烦，执行count(*)的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数。
> - 注意：没有where过滤条件，否则MyISAM也不能返回这么快的

## 为什么InnoDB不跟MyISAM一样，也把数字存起来呢？

> 跟事务有关
>
> 不同的count()用法：
>
> - count()是一个聚合函数，对于返回的结果集，一行行地判断，如果count函数的参数不是NULL，累积值就加1，否则不加。最后返回累计值。
> - count（主键id）
> - count（1）
> - count（字段）
> - count（*）
> - 按照效率排序：![image-20200922170502323](/Users/zachary/Library/Application Support/typora-user-images/image-20200922170502323.png)
> - 建议尽量使用count（*）