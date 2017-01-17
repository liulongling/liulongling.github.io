---
layout:     post
title:      "阿里巴巴Java开发手册"
subtitle:   ""
date:       2017-01-17 18:00:00
author:     "donkey"
header-img: "img/in-post/firewatch.jpg"
tags:
    - Java编程规范
---


> 1.不要嫌名字长

无论是方法，变量，还是函数的取名，不要嫌弃名称太长，只要能够表示清楚含义就可以了。

> 2.String[] args而不是String args[]，中括号是数组类型的一部分，数组定义如下：String[] args;

在《Thinking in Java》这边书里面，是这么解释的：
大部分开发人员，习惯前一种写法。
前一种写法符合我们的口语化，我们口语通常都说：定义一个字符串数组（String 代码字符，[]代表数组）

> 3.POJO 类中的任何布尔类型的变量,都不要加is

POJO类中的任何布尔类型的变量，都不要加 is，否则部分框架解析会引起序列化错 
误。

大部分经典的书籍都有提到这一点，老话题了。

> 4.各层命名规约

A) Service/DAO 层方法命名规约 
1） 获取单个对象的方法用 get 做前缀。 
2） 获取多个对象的方法用 list 做前缀。 
3） 获取统计值的方法用 count 做前缀。 
4） 插入的方法用 save（推荐）或 insert 做前缀。 
5） 删除的方法用 remove（推荐）或 delete 做前缀。 
6） 修改的方法用 update 做前缀。

这一点确实很重要，有时候自己都做不到，规范大于编码。

> 5.不允许出现任何魔法值（即未经定义的常量）

String key="Id#taobao_"+tradeId；
cache.put(key, value);
魔法数值：是指在代码中直接出现的数值，而只有在这个数值记述的那部分代码中才能明确了解其含义。

在这里进行扩充下：

魔法数字的例子


```
int priceTable[] = new int[16]; //ERROR：这个16究竟有何含义呢?
```

使用了带名字的数值的例子

解决方法：

```
static final int PRICE_TABLE_MAX = 16; //OK：带名字
int price Table[] = new int [PRICE_TABLE_MAX]; //OK：名字的含义是很清楚的
```


> 6.多个参数逗号后边必须加空格

有时候做项目，心急如焚时，就会忘记下例中实参的”a”,后边必须要有一个空格

```
method("a", "b", "c");
```

> 7.开发工具编码设置

IDE 的 text file encoding 设置为 UTF-8; IDE 中文件的换行符使用 Unix 格式，不 
要使用 windows 格式

> 8.所有的覆写方法，必须加@Override注解

如果在抽象类中对方法签名进行修改，其实现类会马上编译报错.《Effective Java》这本书已经说的清清楚楚了，大家可以去啃啃这一节的内容。

> 9.提倡尽量不用可变参数编程

相同参数类型，相同业务含义，才可以使用Java的可变参数,避免使用 Object。可变参数必须放置在参数列表的最后


```
public User getUsers(String type, Integer... ids);
```

10.基本数据类型与包装数据类型的使用标准

- 所有的POJO类属性必须使用包装数据类型。 
- RPC方法的返回值和参数必须使用包装数据类型。 
- 所有的局部变量推荐使用基本数据类型。 
说明：POJO类属性没有初值是提醒使用者在需要使用时，必须自己显式地进行赋值，任何NPE 问题，或者入库检查，都由使用者来保证。 

```
正例：数据库的查询结果可能是 null，因为自动拆箱，用基本数据类型接收有 NPE 风险。
```

```
反例：某业务的交易报表上显示成交总额涨跌情况，即正负 x%，x 为基本数据类型，调用的
```

RPC 服务，调用不成功时，返回的是默认值，页面显示：0%，这是不合理的，应该显示成中划线-。所以包装数据类型的null值，能够表示额外的信息，如：远程调用失败，异常退出

> 11.序列化类新增属性时，请不要修改serialVersionUID字段

避免反序列失败

> 12.构造方法里面禁止加入任何业务逻辑

如果有初始化逻辑，请放在init方法中,其实这是符合一个方法只做一件事的原则。

> 13.POJO 类必须写 toString 方法

使用工具类 source> generate toString 时，如果继承了另一个 POJO 类，注意在前面加一下 super.toString。说明：在方法执行抛出异常时，可以直接调用 POJO 的 toString()方法打印其属性值，便于排查问题。

我们项目中，这一点没有做到...

> 14.同名的或者相似的方法放置在一起

> 15.类内方法定义顺序

类内方法定义顺序依次是：公有方法或保护方法 > 私有方法 > getter/setter 方法。

说明： 公有方法是类的调用者和维护者最关心的方法，首屏展示最好；保护方法虽然只是子类 
关心，也可能是“模板设计模式”下的核心方法；而私有方法外部一般不需要特别关心，是一 
个黑盒实现；因为方法信息价值较低，所有 Service 和 DAO 的 getter/setter 方法放在类体最 
后。

> 16.final可提高程序响应效率

final可提高程序响应效率，声明成final的情况： 
- 不需要重新赋值的变量，包括类属性、局部变量。 
- 对象参数前加final，表示不允许修改引用的指向。 
- 类方法确定不允许被重写

> 17.Map/Set的key为自定义对象时，必须重写hashCode和equals

String 重写了 hashCode 和 equals 方法，所以我们可以非常愉快地使用String对象作 为 key 来使用。

> 18.使用集合转数组的方法，必须使用集合的toArray(T[]array)传入的是类型完全一样的数组，大小就是 list.size()。

反例：直接使用 toArray无参方法存在问题，此方法返回值只能是Object[]类，若强转其它类型数组将出现 ClassCastException 错误。

正例：

```
List<String> list = new ArrayList<String>(2); 
list.add("guan"); 
list.add("bao"); 
String[] array = new String[list.size()]; 
array = list.toArray(array);
```

说明：使用 toArray 带参方法，入参分配的数组空间不够大时，toArray 方法内部将重新分配内存空间，并返回新数组地址；如果数组元素大于实际所需，下标为[ list.size() ]的数组元素将被置为null，其它数组元素保持原值，因此最好将方法入参数组大小定义与集合元素个数一致

> 19.Arrays.asList()把数组转换成集合时，不能使用其修改集合相关的方法

它的 add/remove/clear 方法会抛出 UnsupportedOperationException 异常。 
说明： asList 的返回对象是一个 Arrays 内部类，并没有实现集合的修改方法。 Arrays.asList体现的是适配器模式，只是转换接口，后台的数据仍是数组。

```
String[] str = new String[] { "a", "b" };
List  list = Arrays.asList(str);
```

第一种情况：list.add("c"); 运行时异常。
第二种情况：str[0]= "gujin"; 那么 list.get(0)也会随之修改。

> 20.不要在foreach循环里进行元素的remove/add操作

remove元素请使用Iterator方式，如果并发操作，需要对 Iterator 对象加锁。

反例： 

```
List a = new ArrayList(); 
a.add(“1”); 
a.add(“2”); 
for (String temp : a) { 
    if(“1”.equals(temp)){ 
        a.remove(temp); 
    } 
}
```

> 21.集合初始化时，尽量指定集合初始值大小

ArrayList 尽量使用 ArrayList(int initialCapacity) 初始化。

> 22.使用entrySet遍历Map类集合KV，而不是keySet

keySet 其实是遍历了 2 次，一次是转为 Iterator 对象，另一次是从 hashMap 中取出 key所对应的 value。而 entrySet 只是遍历了一次就把 key 和 value 都放到了 entry 中，效率更高。如果是 JDK8，使用Map.foreach 方法。

这一点可以参考《Thinking in Java》第17章容器深入研究这一章

> 23.利用Set元素唯一的特性，可以快速对另一个集合进行去重操作

避免使用List的contains 方法进行遍历去重操作。

> 24.数据库合适的字符存储长度

不但节约数据库表空间、节约索引存储，更重要的是提升检索速度。

> 25.在表查询中，一律不要使用 * 作为查询的字段列表

需要哪些字段必须明确写明。
说明：1）增加查询分析器解析成本。2）增减字段容易与resultMap配置不一致。

> 26.in操作能避免则避免

若实在避免不了，需要仔细评估 in 后边的集合元素数量，控制在 1000 个之内。

> 27.异常信息应该包括两类信息

异常信息应该包括两类信息：案发现场信息和异常堆栈信息。如果不处理，那么往上 
抛。logger.error(各类参数或者对象 toString + "_" + e.getMessage(), e);

> 28.if/else/for/while/do 语句中必须使用大括号，即使只有一行代码

这一点我有点不支持，比如下面的代码：

```
//阿里巴巴建议的写法
if(a > 0){
    System.out.println(".....");
}
```

```
//对于if逻辑中只有一句话，我习惯下面这么些
if(a > 0) System.out.println(".....");
```

那个比较好自己对比吧！！！

> 29.禁止使用存储过程

存储过程难以调试和扩展，更没有移植性


> 30.推荐尽量少用 else， if-else的方式可以改写成

if(condition){ 
    …
    return obj; 
} 


> 31.不要在条件判断中执行复杂的语句


```
正例：

//伪代码如下 
InputStream stream = file.open(fileName, "w");  

if (stream != null) {
    …
}
```

```
反例： 
if (file.open(fileName, “w”) != null)) { 
… 
}
```

> 32.线程安全相关的类使用

SimpleDateFormat 是线程不安全的类，一般不要定义为 static 变量，如果定义为static，必须加锁，或者使用 DateUtils 工具类。


```
正例：注意线程安全，使用 DateUtils。亦推荐如下处理：

private static final ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() { 
    @Override 
    protected DateFormat initialValue() { 
        return new SimpleDateFormat("yyyy-MM-dd"); 
    } 
};
```

说明：如果是 JDK8 的应用，可以使用 instant 代替 Date，Localdatetime 代替 Calendar，Datetimeformatter 代替 Simpledateformatter，官方给出的解释：simple beautiful strong immutable thread-safe。

避免 Random 实例被多线程使用，虽然共享该实例是线程安全的，但会因竞争同一seed 
导致的性能下降。 
说明：Random 实例包括 java.util.Random 的实例或者 Math.random()实例。 
正例：在 JDK7 之后，可以直接使用 API ThreadLocalRandom，在JDK7之前，可以做到每个线程一个实例。

> 33.类、类属性、类方法的注释必须使用 javadoc 规范

类、类属性、类方法的注释必须使用 javadoc 规范，使用/*内容/格式，不得使用//xxx 方式。

方法内部单行注释，在被注释语句上方另起一行，使用//注释。方法内部多行注释使 
用/* */注释，注意与代码对齐

可以参考《代码整洁之道》这本书中的注释这一章，里面内容更加纤细。

> 34.删除被注释掉的没用的东西

对于“明确停止使用的代码和配置”，如方法、变量、类、配置文件、动态配置属性 
等要坚决从程序中清理出去，避免造成过多垃圾。清理这类垃圾代码是技术气场，不要有这样 
的观念：“不做不错，多做多错”。

> 35.获取当前毫秒数：System.currentTimeMillis()

而不是 new Date().getTime();
说明：如果想获取更加精确的纳秒级时间值，用 System.nanoTime。在 JDK8 中，针对统计时间等场景，推荐使用 Instant 类。

【强制】线程资源必须通过线程池提供，不允许在应用中自行显式创建线程。 
说明： 使用线程池的好处是减少在创建和销毁线程上所花的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者“过度切换”的问题。

> 36.所有的类都必须添加创建者信息。

我们元气项目组成员都没有加这个，有的也只是用@author Administrator

> 37.代码修改的同时，注释也要进行相应的修改

尤其是参数、返回值、异常、核心逻辑等的修改。

> 38.任何数据结构的使用都应限制大小

说明：这点很难完全做到，但很多次的故障都是因为数据结构自增长，结果造成内存被吃光。

> 39.日志文件推荐至少保存 15 天

因为有些异常具备以“周”为频次发生的特点。

> 40.对trace/debug/info级别的日志输出，必须使用条件输出形式或者使用占位符的方式

说明：logger.debug(“Processing trade with id: ” + id + ” symbol: ” + symbol); 如果 
日志级别是 warn，上述日志不会打印，但是会执行字符串拼接操作，如果 symbol 是对象，会执行 toString()方法，浪费了系统资源，执行了上述操作，最终日志却没有打印。


```
正例：（条件）

if (logger.isDebugEnabled()) { 
    logger.debug("Processing trade with id: " + id + " symbol: " + symbol); 
} 
正例：（占位符）

    logger.debug("Processing trade with id: {} and symbol : {} ", id,symbol);
```


> 41.特殊注释标记，请注明标记人与标记时间

注意及时处理这些标记，通过标记扫描，经常清理此类标记。线上故障有时候就是来源于这些标记处的代码。

- 待办事宜（TODO）:（ 标记人，标记时间，[预计处理时间]） 
表示需要实现，但目前还未实现的功能。这实际上是一个 javadoc 的标签，目前的 
javadoc 还没有实现，但已经被广泛使用。 只能应用于类，接口和方法（因为它是一个 javadoc 
标签）。
- 错误，不能工作（FIXME）:（标记人，标记时间，[预计处理时间]） 
在注释中用 FIXME 标记某代码是错误的，而且不能工作，需要及时纠正的情况。

> 42.避免用Apache Beanutils 进行属性的 copy。

说明：Apache BeanUtils 性能较差，可以使用其他方案比如 spring BeanUtils, Cglib BeanCopier。

> 43.注意HashMap的扩容死链

导致 CPU 飙升的问题

> 44.应用中的扩展日志（如打点、临时监控、访问日志等）命名方式

appName_logType_logName.log。logType:日志类型，推荐分类有stats/desc/monitor/visit 等；logName:日志描述。这种命名的好处：通过文件名就可知道日志文件属于什么应用，什么类型，什么目的，也有利于归类查找。

> 45.高并发时，同步调用应该去考量锁的性能损耗

能用无锁数据结构，就不要用锁；能锁区块，就不要锁整个方法体；能用对象锁，就不要用类锁。

> 46.在代码中使用“抛异常”还是“返回错误码”

对于公司外的 http/api开放接口必须使用“错误码”；而应用内部推荐异常抛出；跨应用间 RPC 调用优先考虑使用 Result方式，封装isSuccess、“错误码”、“错误简短信息”。

说明：关于 RPC 方法返回方式使用 Result 方式的理由： 
- 使用抛异常返回方式，调用方如果没有捕获到就会产生运行时错误。 
- 如果不加栈信息，只是 new 自定义异常，加入自己的理解的 error message，对于调用端解决问题的帮助不会太多。如果加了栈信息，在频繁调用出错的情况下，数据序列化和传输的性能损耗也是问题。

> 47.谨慎地记录日志。生产环境禁止输出 debug 日志

有选择地输出 info 日志；如果使用warn来记录刚上线时的业务行为信息，一定要注意日志输出量的问题，避免把服务器磁盘撑爆，并记得及时删除这些观察日志。

说明：大量地输出无效日志，不利于系统性能提升，也不利于快速定位错误点。纪录日志时请 

> 48.单表行数超过 500 万行或者单表容量超过 2GB，才推荐进行分库分表。