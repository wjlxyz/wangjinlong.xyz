---
title: 详解Java8的日期时间API
categories:
    - '技术'
    - 'Java基础'
tags:
    - Date/Time
---


在`JDK1.0`的时候，Java引入了`java.util.Date`来处理日期和时间；在`JDK1.1`的时候又引入了功能更强大的`java.util.Calendar`，但是`Calendar`的API还是不尽如人意，，存在实例易变、没有处理闰秒等等的问题。所以在`JDK1.8`的时候，Java引入了`java.time`API，这才真正修改了过去的缺陷，且更为好用。本篇就详细介绍一下`JDK1.8`的日期和时间API。
本篇主要包括以下内容：

- Java8之前的日期和时间API的缺陷
- java.time类图介绍
    - 概况
    - chrono
    - format
    - temporal
    - zone
- Java 8日期/时间类
    - Instant
    - Duration
    - Period
    - LocalDate和LocalTime
    - LocalDateTime
- 日期操作和格式化
- 时区

<!--more-->

### Java8之前的日期和时间API的缺陷
在Java 8之前，所有关于时间和日期的API都存在各种使用方面的缺陷，主要有：
1. Java的java.util.Date和java.util.Calendar类易用性差，不支持时区，而且他们都不是线程安全的；
2. 用于格式化日期的类DateFormat被放在java.text包中，它是一个抽象类，所以我们需要实例化一个SimpleDateFormat对象来处理日期格式化，并且DateFormat也是非线程安全，这意味着如果你在多线程程序中调用同一个DateFormat对象，会得到意想不到的结果。
3. 对日期的计算方式繁琐，而且容易出错，因为月份是从0开始的，从Calendar中获取的月份需要加一才能表示当前月份。
由于以上这些问题，出现了一些第三方的日期处理框架，例如Joda-Time，date4j等开源项目。但是，Java需要一套标准的用于处理时间和日期的框架，于是Java 8中引入了新的日期API。新的日期API是JSR-310规范的实现，Joda-Time框架的作者正是JSR-310的规范的倡导者，所以能从Java 8的日期API中看到很多Joda-Time的特性。

### java.time类图介绍
#### 概况
首先来看一下java.time这个包下的类结构图：
![](/pictures/java.time包图.png)
可以看到，除了一些日期、时间类之外，还有四个包：chrono、format、temporal、zone。
先简略介绍下这四个包的用途。
#### chrono
chrono包提供历法相关的接口与实现。
Java中默认使用的历法是ISO 8601日历系统，它是世界民用历法，也就是我们所说的公历。平年有365天，闰年是366天。闰年的定义是：非世纪年，能被4整除；世纪年能被400整除。为了计算的一致性，公元1年的前一年被当做公元0年，以此类推。
此外chrono包提供了四种其他历法，每种历法有自己的纪元（Era）类、日历类和日期类，分别是：
- 泰国佛教历：ThaiBuddhistEra、ThaiBuddhistChronology和ThaiBuddhistDate；
- 民国历：MinguoEra、MinguoChronology和MinguoDate；
- 日本历：JapaneseEra、JapaneseChronology和JapaneseDate
- 伊斯兰历：HijrahEra、HijrahChronology和HijrahDate：
每个纪元类都是一个枚举类，实现Era接口。Era表示的是一个时间线的分割，比如Java默认的ISO历法中的IsoEra，就包含两个枚举量：BCE和CE，前者表示“公元前”，后者表示“公元”；再比如MinguoEra，包含了两个枚举量：BEFORE_ROC和ROC，ROC的意思是Republic of China，也即新中国，前者表示的就是新中国之前，也即民国，后者表示新中国；所以中国的历法用了“Minguo”这个名字。
每种历法的日历系统的实现都是依赖于其纪元的。每个日历类都实现了抽象类AbstractChronology，其中定义了从时间、id、地域设置获取具体日历系统的接口和实现，以及获取特定日历系统下的时间的方法。
定义了纪元和日历系统之后，日期类自然就确定好了，每种历法的日期类提供的接口并无大的不同，在实际开发中应用的比较少，也不是本篇的重点，暂且略过。
#### format
format包提供了日期格式化的方法。
format包中定义了时区名称、日期解析和格式化的各种枚举，以及最为重要的格式化类DateTimeFormatter。需要注意的是，format包类中的类都是final的，都提供了线程安全的访问。
在DateTimeFormatter类中提供了ofPattern的静态方法来获得一个DateTimeFormatter，但细看其实现，其实还是调用的DateTimeFormatterBuilder的静态方法：`DateTimeFormatterBuilder.appendPattern(pattern).toFormatter();`
所以我们在实际格式化日期和时间的时候，是两种方式都可以使用的。
#### temporal
temporal包中定义了整个日期时间框架的基础：各种时间单位、时间调节器，以及在年月日时分秒中用到的各种属性。
Java8中的日期时间类都是实现了temporal包中的时间单位（Temporal）、时间调节器（TemporalAdjuster）和各种属性的接口，所以在后面的日期的操作方法中都是以最基本的时间单位和各种属性为参数的。
#### zone
这个包没啥多说的，就是定义了时区转换的各种方法。

### Java 8日期/时间类
Java 8的日期和时间类包括Instant、Duration、Period、LocalDate、LocalTime，这些类都包含在java.time包中。下面逐一来看看这些类的用法。
#### Instant
Instant是时间线上的一个点，表示一个时间戳。Instant可以精确到纳秒，这超过了long的最大表示范围，所以在Instant的实现中是分成了两部分来表示，一部分是`seconds`，表示从1970-01-01 00:00:00开始到现在的秒数，另一个部分是`nanos`，表示纳秒部分。
以下是创建Instant的两种方法：
```java:n
Instant now = Instant.now();
Instant instant = Instant.ofEpochSecond(60, 100000);
```
1. 获取当前时刻的时间戳，结果为：`2020-02-20T14:14:15.913Z`；
2. ofEpochSecond()方法的第一个参数为秒，第二个参数为纳秒，上面的代码表示从1970-01-01 00:00:00开始后一分钟的10万纳秒的时刻，其结果为：`1970-01-01T00:01:00.000100Z`。
#### Duration
有了时间点，自然就衍生出时间段了，那就是Duration。Duration的内部实现与Instant类似，也是包含两部分：`seconds`表示秒，`nanos`表示纳秒。
Duration是两个时间戳的差值，所以使用java.time中的时间戳类，例如Instant、LocalDateTime等实现了Temporal类的日期时间类为参数，通过Duration.between()方法创建Duration对象：
```java:n
LocalDateTime from = LocalDateTime.of(2020, Month.JANUARY, 22, 16, 6, 0);    // 2020-01-22 16:06:00
LocalDateTime to = LocalDateTime.of(2020, Month.FEBRUARY, 22, 16, 6, 0);     // 2020-02-22 16:06:00
Duration duration = Duration.between(from, to);     // 表示从 2020-01-22 16:06:00到 2020-02-22 16:06:00 这段时间
```
Duration对象还可以通过of()方法创建，该方法接受一个时间段长度，和一个时间单位作为参数：
```java:n
Duration duration1 = Duration.of(5, ChronoUnit.DAYS);       // 5天
Duration duration2 = Duration.of(1000, ChronoUnit.MILLIS);  // 1000毫秒
```
#### Period
Period在概念上和Duration类似，区别在于Period是以年月日来衡量一个时间段，比如1年2个月3天：
`Period period = Period.of(1, 2, 3);`
Period对象也可以通过between()方法创建，值得注意的是，由于Period是以年月日衡量时间段，所以between()方法只能接收LocalDate类型的参数：
```java:n
// 2020-01-22 到 2020-02-22 这段时间
Period period = Period.between(
                LocalDate.of(2020, 1, 22),
                LocalDate.of(2020, 2, 22));
```
#### LocalDate和LocalTime
LocalDate类表示一个具体的日期，但不包含具体时间，也不包含时区信息。可以通过LocalDate的静态方法of()创建一个实例，LocalDate也包含一些方法用来获取年份，月份，天，星期几等：
```java:n
LocalDate localDate = LocalDate.of(2020, 2, 22);     // 初始化一个日期：2022-02-22
int year = localDate.getYear();                     // 年份：2020
Month month = localDate.getMonth();                 // 月份：February
int dayOfMonth = localDate.getDayOfMonth();         // 月份中的第几天：22
DayOfWeek dayOfWeek = localDate.getDayOfWeek();     // 一周的第几天：Saturday
int length = localDate.lengthOfMonth();             // 月份的天数：29
boolean leapYear = localDate.isLeapYear();          // 是否为闰年：true
```
也可以调用静态方法now()来获取当前日期：`LocalDate now = LocalDate.now();`
LocalTime和LocalDate类似，他们之间的区别在于LocalDate不包含具体时间，而LocalTime包含具体时间，例如：
```java:n
LocalTime localTime = LocalTime.of(16, 14, 52);     // 初始化一个时间：16:14:52
int hour = localTime.getHour();                     // 时：16
int minute = localTime.getMinute();                 // 分：14
int second = localTime.getSecond();                 // 秒：52
```
#### LocalDateTime
LocalDateTime类是LocalDate和LocalTime的结合体，可以通过of()方法直接创建，也可以调用LocalDate的atTime()方法或LocalTime的atDate()方法将LocalDate或LocalTime合并成一个LocalDateTime：
```java:n
LocalDateTime ldt1 = LocalDateTime.of(2020, Month.FEBRUARY, 22, 16, 23, 12);

LocalDate localDate = LocalDate.of(2020, Month.FEBRUARY, 22);
LocalTime localTime = LocalTime.of(16, 23, 12);
LocalDateTime ldt2 = localDate.atTime(localTime);
```
LocalDateTime也提供用于向LocalDate和LocalTime的转化：
```java:n
LocalDate date = ldt1.toLocalDate();
LocalTime time = ldt1.toLocalTime();
```

### 日期操作和格式化
在上面对java.time包中的类的介绍中已经提到，Java8的的日期和时间类都实现了Temporal、TemporalAdjuster，然后在temporal包中定义了日期操作的方法，在format中定义了日期格式化的方法，由此实现了比较通用的日期操作和格式化的方式。
首先需要再次明确的一点是，Java8中提供的日期时间对象都是不可变的，因而也是线程安全的。所以每次对日期时间对象进行操作的时候都是返回新的日期时间对象。
比较简单的日期操作，比如增加、减少一天、修改年月日等，代码如下：
```java:n
LocalDate date = LocalDate.of(2020, 2, 22);          // 2020-02-22
LocalDate date1 = date.withYear(2021);              // 修改为 2021-02-22
LocalDate date2 = date.withMonth(3);                // 修改为 2020-03-22
LocalDate date3 = date.withDayOfMonth(1);           // 修改为 2020-02-01
LocalDate date4 = date.plusYears(1);                // 增加一年 2021-02-22
LocalDate date5 = date.minusMonths(2);              // 减少两个月，到2019年的12月  2019-12-22
LocalDate date6 = date.plus(5, ChronoUnit.DAYS);    // 增加5天 2020-02-27
```
比较复杂的日期操作，比如将时间调到下一个工作日，或者是下个月的最后一天，这时候我们可以使用with()方法的另一个重载方法，它接收一个TemporalAdjuster参数，可以使我们更加灵活的调整日期：
```java:n
LocalDate date7 = date.with(TemporalAdjusters.nextOrSame(DayOfWeek.SUNDAY));      // 返回下一个距离当前时间最近的星期日 2020-02-23
LocalDate date9 = date.with(TemporalAdjusters.lastInMonth(DayOfWeek.SATURDAY));  // 返回本月最后衣蛾周六 2020-02-29
```
下面列出时间调节器类TemporalAdjuster提供的一些方法，可供选用：
```table
方法名 | 描述
dayOfWeekInMonth | 返回同一个月中每周的第几天
firstDayOfMonth | 返回当月的第一天
firstDayOfNextMonth | 返回下月的第一天
firstDayOfNextYear | 返回下一年的第一天
firstDayOfYear	 | 返回本年的第一天
firstInMonth | 返回同一个月中第一个星期几
lastDayOfMonth	 | 返回当月的最后一天
lastDayOfNextMonth | 返回下月的最后一天
lastDayOfNextYear | 返回下一年的最后一天
lastDayOfYear | 返回本年的最后一天
lastInMonth | 返回同一个月中最后一个星期几
next / previous | 返回后一个/前一个给定的星期几
nextOrSame / previousOrSame | 返回后一个/前一个给定的星期几，如果这个值满足条件，直接返回
```
日期格式化的用法请看上面对format包的介绍。

### 时区
对时区处理的优化也是Java8中日期时间API的一大亮点。之前在业务中是真的遇到过一些奇葩的时区问题，在旧的java.util.TimeZone提供的时区不全不说，操作还非常繁琐。
新的时区类java.time.ZoneId是原有的java.util.TimeZone类的替代品。ZoneId对象可以通过ZoneId.of()方法创建，也可以通过ZoneId.systemDefault()获取系统默认时区：
```java:n
ZoneId shanghaiZoneId = ZoneId.of("Asia/Shanghai");
ZoneId systemZoneId = ZoneId.systemDefault();
```
of()方法接收一个“区域/城市”的字符串作为参数，你可以通过getAvailableZoneIds()方法获取所有合法的“区域/城市”字符串：
```java:n
Set<String> zoneIds = ZoneId.getAvailableZoneIds();
```
对于老的时区类TimeZone，Java 8也提供了转化方法：
```java:n
ZoneId oldToNewZoneId = TimeZone.getDefault().toZoneId();
```
有了ZoneId，我们就可以将一个LocalDate、LocalTime或LocalDateTime对象转化为ZonedDateTime对象：
```java:n
LocalDateTime localDateTime = LocalDateTime.now();
ZonedDateTime zonedDateTime = ZonedDateTime.of(localDateTime, shanghaiZoneId);
```
将zonedDateTime打印到控制台为：
```text:n
2020-02-22T16:50:54.658+08:00[Asia/Shanghai]
```
ZonedDateTime对象由两部分构成，LocalDateTime和ZoneId，其中2020-02-22T16:50:54.658部分为LocalDateTime，+08:00[Asia/Shanghai]部分为ZoneId。
另一种表示时区的方式是使用ZoneOffset，它是以当前时间和世界标准时间（UTC）/格林威治时间（GMT）的偏差来计算，例如：
```java:n
ZoneOffset zoneOffset = ZoneOffset.of("+09:00");
LocalDateTime localDateTime = LocalDateTime.now();
OffsetDateTime offsetDateTime = OffsetDateTime.of(localDateTime, zoneOffset);
```

以上就是Java8中关于日期和时间API的内容了。

