> Java8之前，处理日期时间需求时，使用Date、Calender和SimpleDateFormat，来声明时间戳、使用日历处理日期和格式化解析日期时间。但是，这些类的API的缺点比较明显，比如可读性差、易用性差、使用起来冗余繁琐，还有线程安全问题。
>
> Java8推出了新的日期时间类。每一个类功能明确清晰、类之间写作简单、API定义清晰不踩坑，API功能强大无需借助外部工具类即可完成操作，并且线程安全。

### 初始化日期时间

> Data - new Date(2019-1900, 11, 13, 11, 12, 13)
>
> Calendar - Calender.set(2019, 11, 31, 11, 12, 13)

### 恼人的时区问题

> **关于Date类的认知**：
>
> - Date并无时区问题，世界上任何一台计算机使用new Date()初始化得到的时间都一样。因为，Date中保存的是UTC时间，UTC是以原子钟为基础的统一时间，不以太阳参照计时，并无时区划分
> - Date中保存的是一个时间戳，代表的是从1970年1月1日0点（Epoch时间）到现在的毫秒数。尝试输出Date(0)
>
> 对于国际化项目，处理好时间和时区问题首先就是要正确保存日期时间。这里有两种保存方式：
>
> - 以UTC保存，保存的时间没有时区属性，是不涉及时区时间差问题的世界统一时间。通常说的时间戳，或Java中的Date类就是用的这种方式，这也是推荐的方式。
> - 以字面量保存，比如年/月/日 时：分：秒，一定要同时保存时区信息。只有有了时区信息，才能知道字面量时间真正的时间点，否则只是一个给人看的时间表示，只在当前时区有意义。Calendar是有时区概念的，所有通过不同的时区初始化Calendar，得到了不同的时间
>
> 从字面量解析成时间和从时间格式化为字面量这两类问题：
>
> - 对于同一个时间表示，比如2020-01-02 22:00:00，不同时区的人转换成Date会得到不同的时间（时间戳）。
> - 格式化后出现的错乱，即同一个Date，在不同的时区下格式化得到不同的时间表示。
>
> 要正确处理国际化时间问题，推荐使用Java8的日期时间类：
>
> ```java
>  //一个时间表示
>     String stringDate = "2020-01-02 22:00:00";
>     //初始化三个时区
>     ZoneId timeZoneSH = ZoneId.of("Asia/Shanghai");
>     ZoneId timeZoneNY = ZoneId.of("America/New_York");
>     ZoneId timeZoneJST = ZoneOffset.ofHours(9);
>     //格式化器
>     DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
>     ZonedDateTime date = ZonedDateTime.of(LocalDateTime.parse(stringDate, dateTimeFormatter), timeZoneJST);
>     //使用DateTimeFormatter格式化时间，可以通过withZone方法直接设置格式化使用的时区
>     DateTimeFormatter outputFormat = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss Z");
>     System.out.println(timeZoneSH.getId() + outputFormat.withZone(timeZoneSH).format(date));
>     System.out.println(timeZoneNY.getId() + outputFormat.withZone(timeZoneNY).format(date));
>     System.out.println(timeZoneJST.getId() + outputFormat.withZone(timeZoneJST).format(date));
> ```



### 使用遗留的SimpleDateFormat会遇到哪些问题

> ***日期时间格式化和解析***
>
> - 年底提前跨年
>   - 小写y是年，大写Y是week year，也就是所在的周属于哪一年
>   - 针对年份的日期格式化，应该一律使用“y”而非“Y”
> - 定义的static的SimpleDateFormat可能会出现线程安全问题。
>   - 其解析和格式化操作是非线程安全的
>     - DateFormat、CalendarBuilder、establish
>     - 只能在同一个线程复用SimpleDateFormat，比较好的解决方式是，通过ThreadLocal来存放SimpleDateFormat
>   - 当需要解析的字符串和格式不匹配的时候，SimpleDateFormat表现的很宽容，还是能得到结果。例如起期望使用yyyyMM 来解析20160901，结果输出了2091年1月1日，原因是把0901当成了月份，相当于75年
> - DateTimeFormatter是线程安全的，可以定义为static使用；DateTimeFormatter的解析比较严格，需要解析的字符串和格式不匹配时，会直接报错。

### 日期时间的计算

> ***int溢出***
>
> ***LocalDate.now()***
>
> ```java
> LocalDateTime.now()
>   .mius(Period.ofDays(1))
>   .plus(1, ChrononUnit.DAYS)
>   .minusMonths(1)
>   .plus(Period.ofMonths(1));
> ```
>
> ***通过with方法进行快捷时间调节，比如：***
>
> - 使用 TemporalAdjusters.firstDayOfMonth 得到当前月的第一天
> - 使用 TemporalAdjusters.firstDayOfYear() 得到当前年的第一天
> - 使用 TemporalAdjusters.previous(DayOfWeek.SATURDAY) 得到上一个周六
> - 使用 TemporalAdjusters.lastInMonth(DayOfWeek.FRIDAY) 得到本月最后一个周五
>
> ***Java :: 语法，调用类或对象的构造器、相关方法***
>
> ***使用Java8操作和计算日期时间虽然方便，但计算两个日期差时可能会踩坑：***
>
> - Java8中有个专门的类Period定义了日期间隔，通过Period.between得到了两个LocalDate的差，返回的是两个日期差几年零几月零几天。如果希望得知两个日期之间差几天，直接调用Period的getDays()方法得到的只是最后的“零几天”，而不是算总的间隔天数。
>
> - 可以使用C h ronoUnit.DAYS.between解决这个问题：
>
> - ``` java
>   System.out.println("//计算日期差");
>   LocalDate today = LocalDate.of(2019, 12, 12);
>   LocalDate specifyDate = LocalDate.of(2019, 10, 1);
>   System.out.println(Period.between(specifyDate, today).getDays());
>   System.out.println(Period.between(specifyDate, today));
>   System.out.println(ChronoUnit.DAYS.between(specifyDate, today));
>   ```
>
> - 
>
> 
>
> 

