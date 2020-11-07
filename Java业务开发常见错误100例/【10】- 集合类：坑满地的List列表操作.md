> 把数组转换为List集合、对List进行切片操作、List搜索的性能问题

> Java的集合类包括Map和Collection两大类。Collection包括List、Set和Queue三个小类，其中List列表集合是最重要也是所有业务代码都会用到的。所以，重点是List。

### 使用Arrays.asList把数据转换为List的三个坑

1. **不能直接使用Arrays.asList来转换基本类型数组**

   > 只能把int装箱为Integer，而不能把int数组装箱为Integer数组。可以知道，Arrays.asList方法传入的是一个泛型T类型可变参数，最终int数组整体作为了一个对象成为了泛型类型T。
   >
   > **修复方式**
   >
   > - Java8 以上版本可以使用Arrays.stream方法来转换
   > - 把int数组声明为包装类型Integer数组

2. **Arrays.asList返回的List不支持增删操作**

   > Arrays.asList返回的List并不是期望的java.util.ArrayList，而是Arrays的内部类。ArrayList内部类继承自AbstractList类，并没有覆写父类的add方法，而父类中add方法的实现，就是抛出UnsupportOperationException

3. **对原始数组的修改会影响到我们获得的那个List**

   > 查看ArrayList的实现，可以发现ArrayList其实是直接使用了原始的数组。所以，要特别小心，把通过Arrays.asList获得的List交给其他方法处理，很容易因为共享了数组，相互修改产生Bug
   >
   > **修复方式**
   >
   > - 重新new一个ArrayList初始化Arrays.asList返回的List即可。修改后的代码实现了原始数组和List的“解耦”，不再相互影响。同时，因为操作的是真正的ArrayList，add也不再出错

### 使用List.subList进行切片操作居然会导致OOM

> 与Arrays.asList的问题类似，List.subList返回的子List不是一个普通的ArrayList。这个字List可以认为是原始List视图，会和原始List相互影响。如果不注意，很可能会因此产生OOM问题。
>
> subList对List依然持有强引用
>
> ```java
> List<Integer> list = IntStream.rangeClosed(1, 10).boxed().collect(Collectors.toList());
> List<Integer> subList = list.subList(1, 4);
> System.out.println(subList);
> subList.remove(1);
> System.out.println(list);
> list.add(0);
> try {
>     subList.forEach(System.out::println);
> } catch (Exception ex) {
>     ex.printStackTrace();
> }
> ```
>
> ```java
> //输出
> [2, 3, 4]
> [1, 2, 4, 5, 6, 7, 8, 9, 10]
> java.util.ConcurrentModificationException
>   at java.util.ArrayList$SubList.checkForComodification(ArrayList.java:1239)
>   at java.util.ArrayList$SubList.listIterator(ArrayList.java:1099)
>   at java.util.AbstractList.listIterator(AbstractList.java:299)
>   at java.util.ArrayList$SubList.iterator(ArrayList.java:1095)
>   at java.lang.Iterable.forEach(Iterable.java:74)
> ```
>
> **结果**
>
> - 原始List中数字3被删除了，说明删除子List中的元素影响到了原始List
> - 尝试为原始List增加数字0之后，再遍历子List，会出现ConcurrentModificationException
>
> **分析**
>
> 1. ArrayList维护了一个叫作modCount的字段，表示集合结构性修改的次数。所谓结构性修改，指的是影响List大小的修改，所以add操作必然会改变modCount的值
> 2. 通过subList方法获得的List其实是内部类SubList，不是普通的ArrayList，在初始化的时候传入了this。
> 3. SubList中的parent字段就是原始的List。SubList初始化的时候，并没有把原始List中的元素复制到独立的变量中保存。可以认为SubList是原始List的视图，并不是独立的List。双方对元素的修改会相互影响，而且SubList强引用了原始的List，所以大量保存这样的SubList会导致OOM。
> 4. 遍历SubList的时候会先获得迭代器，比较原始ArrayList modCount的值和SubList当前modCount的值。获得了SubList后哦，为原始List新增了一个元素修改了其modCount，所以判等失败，抛出ConcurrentModificationException异常。
>
> **修复**
>
> - 不直接使用subList方法返回的SubList，而是重新使用new ArrayList，在构造方法传入SubList，来构建一个独立的ArrayList。
> - 对于Java8使用Stream的skip和limit API来跳过流中的元素，以及限制流中元素的个数，同样可以达到SubList切片的目的。

### 一定要让合适的数据结构做合适的事情

> mapToObj
>
> ObjectSizeCalculator 只有jdk1.8有？
>
> - **误区1**： 使用数据结构不考虑平衡时间和空间
> - **误区2**： 过于迷信教科书的大O时间复杂度，其实LinkedList随机访问和插入性能都比较差
>
> 查看LinkedList源码发现，插入操作的时间复杂度是O(1)的前提是i，已经有了要插入节点的指针。前提也是有开销的，不能只考虑插入操作本身的代价。

# 总结

1. 认为Arrays.asList和List.subList得到的List是普通的、独立的ArrayList，在使用时出现各种奇怪的问题
   - Arrays.asList得到的是Arrays的内部类ArrayList，List.subList得到的是ArrayList的内部类SubList，不能把这两个内部类转换为ArrayList使用
   - Arrays.asList直接使用了原始数组，可以认为是共享“存储”，而且不支持增删元素；List.subList直接引用了原始的List，也可以认为是共享“存储”，而且对原始List直接进行结构性修改会导致SubList出现异常
   - 对Arrays.asList和List.subList容易忽略的是，新的List持有了原始数据的引用，可能会导致原始数据也无法GC的问题，最终导致OOM
2. 认为Arrays.asList一定可以把所有数组转换为正确的List。当传入基本类型数组的时候，List的元素是数组本身，而不是数组中的元素。
3. 认为，内存中任何集合的搜索都是很快的，结果在搜索超大ArrayList的时候遇到性能问题。考虑利用HashMap哈希表随机查找的时间复杂度为O(1)这个特性来优化性能，不过也要考虑HashMap存储空间上的代价，要平衡时间和空间
4. 认为，连标适合元素增删的场景，选用LinkedList作为数据结构。在真实场景中读写增删一般是平衡的，而且增删不可能只是对头尾对象进行操作，可能在90%的情况下都得不到性能增益，建议使用之前通过性能测试评估一下。





