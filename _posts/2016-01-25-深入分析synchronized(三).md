---
layout:     post
title:      "深入分析synchronized(三)"
subtitle:   ""
date:       2016-01-25 15:00:00
author:     "donkey"
header-img: "img/in-post/firewatch.jpg"
tags:
    - Java并发编程
---

# 一、synchronized简介

Java提供了强制性的锁机制：synchronized，可用来给对象和方法或者代码块加锁，当它锁定一个方法或者一个代码块的时候，同一时刻最多只有一个线程执行这段代码。当两个并发线程访问同一个对象object中的这个加锁同步代码块时，一个时间内只能有一个线程得到执行。另一个线程必须等待当前线程执行完这个代码块以后才能执行该代码块。然而，当一个线程访问object的一个加锁代码块时，另一个线程仍然可以访问该object中的非加锁代码块。

包括两种用法：synchronized方法和 synchronized 块。

# 二、synchronized的使用

## 2.1 synchronized方法

```
public static synchronized void inc() {  
    ....  
}  
```

 synchronized方法控制多个线程对该方法内的成员的并发访问，我们将inc方法申明为synchronized,所以同一时间只有一个线程可以访问inc方法。当第一个玩家进来后，会对inc方法加一把锁不让别的玩家访问，玩家二如果也想访问inc方法只能排队等候直到玩家一访问结束释放锁后，效果如下图。 虽然它现在是线程安全的了，但是这种方法过于极端，它的性能非常差。因为有时候我们需要共享的只是方法内的部分数据，其它数据是可以自由访问的，那么这个时候我们应该在项目中使用synchronized块。
 
 
![image](http://img.blog.csdn.net/20160722171959096?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##  2.2 synchronized代码块

 synchronized代码块控制线程访问的数据在synchronized(obj)或synchronized(this){}里面，同一时间也只能有一个线程可以访问，别的请求线程将被阻塞在 synchronized(obj){}外边，这样可以不影响别的线程访问不需要共享的数据。比如：inc() 玩家等级小于30 ，不满足条件程序直接return不用让线程也阻塞在synchronized代码块外边。
 
 
```
public void inc(Object obj) {  
    if(obj == null)  
    {  
       return;  
    }  
    synchronized (obj) {  
        count++;  
    }  
} 
```


```
public void inc(int lvl) {  
       if(lvl < 30)//玩家等级小于30 返回  
    {  
        return;  
    }  
    synchronized (this) {  
        count++;  
    }  
}  
```

##  2.3 对synchronized(this)的理解

- 当两个并发线程访问同一个对象object中的这个synchronized(this)同步代码块时，一个时间内只能有一个线程得到执行。另一个线程必须等待当前线程执行完这个代码块以后才能执行该代码块。

- 然而，另一个线程仍然可以访问该object中的非synchronized(this)同步代码块。
- 尤其关键的是，当一个线程访问object的一个synchronized(this)同步代码块时，其他线程对object中所有其它synchronized(this)同步代码块的访问将被阻塞。
- 第三个例子同样适用其它同步代码块，它就获得了这个object的对象锁。结果，其它线程对该object对象所有同步代码部分的访问都被暂时阻塞。
- 以上规则对其它对象锁同样适用。

  以上是摘自百度百科对synchronized的理解，详细使用参考这篇文章[：http://www.cnblogs.com/GnagWang/archive/2011/02/27/1966606.html](http://www.cnblogs.com/GnagWang/archive/2011/02/27/1966606.html)


# 三、内部锁的重进入

当一个线程请求其它线程已经占有的锁时，请求线程将被阻塞。然而内部锁是可重进入的，因此线程在试图获得它自己占有的锁时，请求会成功。重进入意味着锁的请求是基于“每线程”，而不是基于“每调用”的。重进入的实现是通过每个锁关联一个请求计数和一个占有它的线程。当计数为0时，认为锁时未被占有的。线程请求一个未被占有的锁时，JVM将记录锁的占有者，并且将请求计数置为1。如果同一线程再次请求这个锁，计数将递增；每次占用线程退出同步块，计数器值将递减。直到计数器达到0时，锁被释放。【摘自JAVA并发编程实战】

##   3.1代码示例


```
package com.game.lll.syn;  
  
import java.util.concurrent.ExecutorService;  
import java.util.concurrent.Executors;  
import java.util.concurrent.TimeUnit;  
  
  
public class UnsafeCount {  
    public static int count = 0;  
    static LoggingWidget loggingWidget = new LoggingWidget();  
    public static void inc() {  
        loggingWidget.doSomething();  
    }  
  
    public static void main(String[] args) throws InterruptedException {  
  
        ExecutorService service=Executors.newFixedThreadPool(Integer.MAX_VALUE);  
  
        for (int i = 0; i < 10; i++) {  
            service.execute(new Runnable() {  
                @Override  
                public void run() {  
                    UnsafeCount.inc();  
                }  
            });  
        }  
  
        service.shutdown();  
        //避免出现main主线程先跑完而子线程还没结束，在这里给予一个关闭时间  
        service.awaitTermination(3000,TimeUnit.SECONDS);  
        System.out.println("运行结果:UnsafeCount.count=" + UnsafeCount.count);  
    }  
}  
```


```
package com.game.lll.syn;  
  
public class LoggingWidget extends Widget{  
    public synchronized void doSomething()  
    {  
        System.out.println("LoggingWidget"+UnsafeCount.count++);  
        super.doSomething();  
    }  
}  
```


```
package com.game.lll.syn;  
public class Widget {  
   public synchronized void doSomething()  
   {  
       System.out.println("Widget"+UnsafeCount.count++);  
   }  
}  
```

控制台输出：


```
LoggingWidget0
Widget1
LoggingWidget2
Widget3
LoggingWidget4
Widget5
LoggingWidget6
Widget7
LoggingWidget8
Widget9
LoggingWidget10
Widget11
LoggingWidget12
Widget13
LoggingWidget14
Widget15
LoggingWidget16
Widget17
LoggingWidget18
Widget19
运行结果:UnsafeCount.count=20
```

## 3.2代码分析

重进入方便了锁行为的封装，因此简化了面向对象并发代码的开发。上面代码子类覆写了父类的synchronized类型的方法，并调用父类中的方法。如果没有可重入的锁，这段代码将会产生死锁。因为Weight和loggingWeight中的soSomething方法都是synchronized类型的，都会在处理前试图获得weight的锁。倘若内部锁不是可重入的，super.doSomething的调用者就永远无法得到weight的锁，因为锁已经被占有，导致线程会永久的延迟，等待着一个永远无法获得的锁。

# 四、锁的三大特性

## 4.1 原子性

>   原子性是指在同一时刻只有一个线程对它进行读写操作，避免多个线程在更改共享数据时出现数据的不准确。
  
  在Java中提供了原子操作的关键字synchronized。我在上一篇文章中写过[原子性Atomic(一)](https://liulongling.github.io/2016/01/20/%E5%8E%9F%E5%AD%90%E6%80%A7Atomic(%E4%B8%80)/)
  
## 4.2 可见性
  
> 可见性是指当一个线程修改了线程共享变量的值，其它线程能够立即得知这个值的修改。在Java中，除了synchronized，volatile和final也是可见性的。synchronized可以确保线程能预见另一个线程对某一个值或状态的更改，就像下图一样。当线程一执行一个同步块时，线程二也随后进入了同一个锁的同步块中，这时可以保证，在释放锁M之前线程一变量的值count对线程二是可见的。换句话说就是有一个玻璃透明的房间，虽然线程一进入后将房间锁住了，但是线程二在门口还是可以透过玻璃看见房间内的一切事物。

![image](http://img.blog.csdn.net/20160723154501333?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


```
package com.game.lll.syn;  
  
import java.util.concurrent.ExecutorService;  
import java.util.concurrent.Executors;  
import java.util.concurrent.TimeUnit;  
  
  
public class SafeCount implements Runnable{  
    public  volatile static int count = 0;  
  
    public synchronized static void inc()  
    {  
        count++;  
    }  
      
    public static void main(String[] args) throws InterruptedException {  
  
         SafeCount t1 = new SafeCount();    
         Thread ta = new Thread(t1, "线程一");    
         Thread tb = new Thread(t1, "线程二");    
         ta.start();    
         tb.start();    
        System.out.println("UnsafeCount.count=" + SafeCount.count);  
    }  
  
    @Override  
    public void run() {  
        System.out.println(Thread.currentThread().getName()+"---执行前--count:"+count);    
        inc();  
        System.out.println(Thread.currentThread().getName()+"---执行后--count:"+count);    
    }  
      
}  

```

控制台输出：


```
UnsafeCount.count=0
线程二---执行前--count:0
线程一---执行前--count:0
线程二---执行后--count:1
线程一---执行后--count:2

```

> 锁不仅仅是关于同步与互斥，也是关于内存可见的。为了保证所有线程看到共享的、可变变量的最新值，读写和写入线程必须使用公共的锁进行同步。摘自--《Java并发编程实战》

## 4.3 有序性

> Java语言提供了volatile和synchronized两个关键字来保证线程之间操作的有序性，volatile关键字本身就包含了禁止指令重排序的语义，而synchronized则是由“一个变量在同一时刻只允许一条线程对其进行lock操作”这条规则来获得的，这个规则决定了持有同一个锁的两个同步块只能串行地进入。

  1.程序次序规则(Pragram Order Rule)：在一个线程内，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作。准确地说应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环结构。

  2.管程锁定规则(Monitor Lock Rule)：一个unlock操作先行发生于后面对同一个锁的lock操作。这里必须强调的是同一个锁，而”后面“是指时间上的先后顺序。。

  3.volatile变量规则(Volatile Variable Rule)：对一个volatile变量的写操作先行发生于后面对这个变量的读取操作，这里的”后面“同样指时间上的先后顺序。

  4.线程启动规则(Thread Start Rule)：Thread对象的start()方法先行发生于此线程的每一个动作。

  5.线程终于规则(Thread Termination Rule)：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread.join()方法结束，Thread.isAlive()的返回值等作段检测到线程已经终止执行。

  6.线程中断规则(Thread Interruption Rule)：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted()方法检测是否有中断发生。

  7.对象终结规则(Finalizer Rule)：一个对象初始化完成(构造方法执行完成)先行发生于它的finalize()方法的开始。

  8.传递性(Transitivity)：如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。
  
详细请参考这篇文章[：深入理解Java虚拟机笔记---原子性、可见性、有序性](http://www.tuicool.com/articles/ru6vUvn)

## 4.4 应用场景

