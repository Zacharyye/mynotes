## 25、JVM内存区域的划分，OutOfMemoryError

> JVM内部的内存结构、工作机制
>
> - 程序计数器 PC，每个线程都有自己的程序计数器，并且任何时间一个线程都只有一个方法在执行，也就是所谓的当前方法。
> - Java虚拟机栈 - Java栈，每个线程在创建时都会创建一个虚拟机栈，其内部保存一个个的栈帧，其内部保存一个个的栈帧，对应着一次次的Java方法调用。
>   - 在一个时间点，对应的只会有一个活动的栈帧，通常叫做当前帧，方法所在的类叫做当前类。如果在该方法中调用了其他方法，对应的新的栈帧会被创建出来，成为新的当前帧，一直到它返回结果或者执行结束。JVM直接对Java栈的操作只有两个，就是对栈帧的压栈和出栈。
>   - 栈帧中存储着局部变量表、操作数栈、动态链接、方法正常退出或者异常退出的定义等。
> - 堆（Heap），是Java内存管理的核心区域，用来放置Java对象实例，几乎所有创建的Java对象实例都是被直接分配在堆上。堆被所有的线程共享，在虚拟机启动时，指定“Xmx”之类参数就是用来指定最大堆空间等指标。
>   - 新生代、老年代
> - 方法区 - 所有线程共享的一块内存区域，用于存储所谓的元（Meta）数据，例如类结构信息，以及对应的运行时常量池、字段、方法代码等。
>   - 早期称为永久代，Oracle JDK8中将永久代移除，同时增加了元数据区
> - 运行时常量池 - 方法区的一部分。
> - 本地方法栈 -  与Java虚拟机栈非常相似，支持对本地方法的调用，也是每个线程都会创建一个

## 知识扩展

> 直接内存（Direct Memory）



# 26 如何监控和诊断JVM堆内和堆外内存使用？

> JConsole、VisualVM

![img](https://static001.geekbang.org/resource/image/72/89/721e97abc93449fbdb4c071f7b3b5289.png)

### 新生代

> 大部分对象创建和销毁的区域，在通常的Java应用中，绝大部分对象生命周期都是很短暂的。其内部又分为Eden区域，作为对象初始分配的区域；两个Survivor，有时候也叫from、to区域，被用来放置从Minor GC中保留下来的对象。
>
> - JVM会随意选取一个Survivor区域作为“to“，然后会在GC过程中进行区域间拷贝，也就是将Eden中存活下来的对象和from区域的对象，拷贝到这个”to“区域。这种设计主要是为了防止内存的碎片化，并进一步清理无用对象。
> - 从内存模型而不是垃圾收集的角度，对Eden区域继续进行划分，Hotspot JVM还有一个概念叫做Thread Local Allocation Buffer（TLAB），据说所有OpenJDK衍生出来的JVM都提供了TLAB的设计。这是JVM为每个线程分配的一个私有缓存区域，否则，多线程同时分配内存时，为避免操作同一地址，可能需要使用加锁等机制，进而影响分配速度，可以参考下面的示意图。从图中可以看出，TLAB仍然在堆上，它是分配在Eden区域内的。其内部结构比较直观易懂，start、end就是起始地址，top（指针）则表示已经分配到哪里了。所以，分配新对象时，JVM就会移动top，当top和end相遇时，即表示该缓存已满，JVM会试图再从Eden里分配一块。
> - ![img](https://static001.geekbang.org/resource/image/f5/bd/f546839e98ea5d43b595235849b0f2bd.png)

### 老年代

> 放置长生命周期的对象，通常都是从Survivor区域拷贝过来的对象。当然，也有特殊情况，我们知道普通的对象会被分配到TLAB上；如果对象较大，JVM会试图直接分配在Eden其他位置上；如果对象太大，完全无法在新生代找到足够长的连续空闲空间，JVM就会直接分配到老年代。

### 永久代

> 这部分就是早期的Hotspot JVM的方法区实现方式了，储存Java类元数据、常量池、Intern字符串缓存，在JDK8之后就不存在永久代这块儿了。



> 那么，如何利用JVM参数，直接影响堆和内部区域的大小呢？
>
> - 最大堆体积`-Xmx value`
> - 初始的最小堆体积`-Xms value`
> - 老年代和新生代的比例`-XX:NewRatio=value`
>   - 默认情况下，这个数值是2，意味着老年代是新生代的2倍大；换句话说，新生代是堆大小的1/3
> - 当然，也可以不用比例的方式调整新生代的大小，直接指定下面的参数，设定具体的内存大小数值`-XX:NewSize=value`
> - Eden和Survivor的大小是按照比例设置的，如果SurvivorRatio是8，那么Survivor区域就是Eden的1/8大小，也就是新生代的1/10.参数格式：`-XX:SurvivorRatio=value` 
> - TLAB当然也可以调整，JVM实现了复杂的适应策略。
> - Virtual区域 - 在JVM内部，如果Xms小于Xmx，堆的大小并不会直接扩展到其上限，也就是说保留的空间大于实际能够使用的空间。当内存需求不断增长的时候，JVM会逐渐扩展新生代等区域的大小，所以Virtual区域代表的就是暂时不可用空间。

