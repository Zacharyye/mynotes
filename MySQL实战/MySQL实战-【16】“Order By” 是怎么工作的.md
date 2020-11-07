# 全字段排序

> explain -> Extra -> Using filesort表示的是需要排序，MySQL会给每个线程分配一块内存用于排序，称为sort_buffer

> ![图片1](https://static001.geekbang.org/resource/image/6c/72/6c821828cddf46670f9d56e126e3e772.jpg)
>
> 按name排序操作，可能在内存中完成，也可能需要使用外部排序，这取决于所需的内存和参数sort_buffer_size。
>
> - ![img](https://static001.geekbang.org/resource/image/53/3e/5334cca9118be14bde95ec94b02f0a3e.png)
>
> - 从图中可以看到，满足city=“杭州”条件的行，是从ID_X到ID_（X+N）的这些记录：
>
>   > - 初始化sort_buffer，确定放入name、city、age这个三个字段
>   > - 从索引city找到第一个满足city=“杭州”条件的主键id，也就是图中的ID_X；
>   > - 到主键id索引取出整行，取name、city、age三个字段的值，存入sort_buffer中；
>   > - 从索引city取下一个记录的主键id
>   > - 重复步骤3、4直到city的值不满足查询条件为止，对应的主键id也就是图中的ID_Y
>   > - 对sort_buffer中的数据按照字段name做快速排序
>   > - 按照排序结果取前1000行返回给客户端
>
> - 全字段排序：可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数sort_buffer_size
>
>   > `sort_buffer_size` 就是MySQL为排序开辟的内存（sort_buffer）的大小。如果要排序的数据量小于sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

# rowid排序

> 在上面的这个算法过程中，只对原表的数据读了一遍，剩下的操作都是在sort_buffer和临时文件中执行的。但这个算法有一个问题，就是如果查询要返回的字段很多的话，那么sort_buffer里面要放的字段数太多，这样内存里能够同时放下的行数很少，要分成很多个临时文件，排序的性能会很差。
>
> ## 如果MySQL认为排序的单行长度太大会怎么做呢？
>
> `max_length_for_sort_data` 是MySQL中专门控制用于排序的行数据的长度的一个参数，意思是：如果单行的长度超过这个值，MySQL就认为单行太大，要换一个算法。
>
> city、name、age这三个字段的定义总长度是36
>
> 新的算法放入sort_buffer的字段，只有要排序的列（即name字段）和主键id。

