> 程序中的变量是null，就意味着它没有引用指向或者说没有指针。这时，对这个变量进行任何操作，都必然会引发空指针异常，在Java中就是NullPointException。那么，空指针异常容易在哪些情况下出现，又应该如何修复？
>
> 空指针异常虽然恼人但好在容易定位，更麻烦的是要弄清楚null的含义，比如，客户端给服务端的一个数据是null，那么其意图到底是给一个空值，还是没提供值呢？再比如，数据库中字段的NULL值，是否有特殊的含义，针对数据库中的NULL值，写SQL需要特别注意什么呢？



### 修复和定位恼人的空指针问题

> **NullPointerException** 是Java代码中最常见的异常，我将其最可能出现的场景归为以下5种：
>
> - 参数值是Integer等包装类型，使用时因为自动拆箱出现了空指针异常
> - 字符串比较出现空指针异常
> - 诸如ConcurrentHashMap这样的容器不支持Key和Value为null，强行put null的Key或Value会出现空指针异常
> - A对象包含了B，在通过A对象的字段获得B之后，没有对字段判空就级联调用B的方法出现空指针异常
> - 方法或远程服务返回的List不是空而是null，没鱼哦进行判空就直接调用List的方法出现空指针异常

### 小心MySQL中关于NULL的三个坑



>  **arthas**  Java故障诊断神器，简单易用，可以定位出大多数的Java生产问题。
>
> 1. 下载arthas-packaging-3.4.3-bin包
> 2. 解压后sh as.sh
> 3. `watch com.zzzz.test2020101601.controller.Test1Controller wrongMethod params` 
> 4. 访问请求，观察参数情况
>
> 至此，如果是简单的业务逻辑的话，可以定位到空指针异常了；如果是分支复杂的业务逻辑，需要再借助stack(_再arthas环境下使用_)命令来查看wrongMethod方法的调用栈，并配合watch命令查看各方法的入参，就可以很方便地定位到空指针的根源了。

**修复**：

> - 对于Integer的判空，可以使用Optional.ofNullable来构造一个Optional，然后使用orElse（0）把null替换为默认值再进行+1操作
> - 对String和字面量的比较，可以把字面量放在前面，比如“OK”.equals(s)，这样即使s是null也不会出现空指针异常；而对于两个可能为null的字符串变量的equals比较，可以使用Objects.equals
> - 对于ConcurrentHashMap，既然其Key和Value都不支持null，修复方式就是不要把null存进去。HashMap的Key和Value可以存入null，而ConcurrentHashMap看似是HashMap的线程安全版本，却不支持null值的Key和Value，这是容易产生误区的地方
> - 对于类似fooService.getBarService().bar().equals("OK")的级联调用，需要判空的地方有很多，包括fooService、getBarService方法的返回值，以及bar方法返回的字符串。如果使用if-else来判空的话可能需要好几行代码，但使用Optional的话一行代码就够了
> - 对于rightMethod返回的List，由于不能确认其是否为null，所以在调用size方法获得列表大小之前，同样可以使用Optional.ofNullable包装一下返回值，然后通过.orElse(Collections.emptyList())实现在List为null的时候哦获得一个空的List，最后再调用size方法

### 使用判空方式或者Optional方式来避免出现指针异常，不一定是解决问题的最好方式，空指针没出现可能隐藏了更深的Bug

### POJO中属性的null到底代表了什么？

> - DTO中字段的null到底意味着什么？
> - 既然空指针很讨厌，那么DTO中的字段要设置默认值么
> - 如果DTO中的字段有null，会覆盖数据库中的既有数据么
>
> **问题**总结：
>
> - 明确DTO中null的含义，对于JSON到DTO的反序列化过程，null的表达是有歧义的，客户端不传某个属性，或者传null，这个属性在DTO中都是null。但，对于用户信息更新操作，不传意味着客户端不需要更新这个属性，维持数据库原先的值；传了null，意味着客户端希望重置这个属性。因为Java中的null就是没有这个数据，无法区分这两种表达，所以本例中的age属性也被设置为了null，或许可以借助Optional来解决这个问题
> - POJO中的字段有默认值。如果客户端不传值，就会赋值为默认值，导致创建时间也被更新到了数据库中。
> - 注意字符串格式化时可能会把null值格式化为null字符串。
> - DTO和Entity共用了一个POJO。对于用户昵称的设置是程序控制的，不应该把它们暴露在DTO中，否则很容易把客户端随意设置的值更新到数据库中。此外，创建时间最好让数据库设置为当前时间，不用程序控制，可以通过在字段上设置columnDefinition来实现。
> - 数据库字段允许保存null，会进一步增加出错的可能性和复杂度。

### 小心MySQL中有关NULL的三个坑

> sum函数、count函数、NULL值条件
>
> - MySQL中sum函数没统计到任何记录时，会返回null而不是0，可以使用IFNULL函数把null转换为0
> - MySQL中count字段不统计null值，COUNT(*)才是统计所有记录数量的正确方式
> - MySQL中使用诸如=、<、>这样的算数比较操作符比较NULL结果总是NULL，需要使用IS NULL、IS NOT NULL或ISNULL函数来比较

