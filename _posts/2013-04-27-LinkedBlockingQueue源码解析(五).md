---
layout:     post
title:      "LinkedBlockingQueue源码分析(五)"
subtitle:   ""
date:       2013-04-27 18:00:00
author:     "donkey"
header-img: "img/in-post/firewatch.jpg"
tags:
    - Java并发编程
---


#1.1 简介
 LinkedBlockingQueue是一个由链表结构组成的有界阻塞队列，此队列是FIFO(先进先出)的顺序来访问的，它由队尾插入后再从队头取出或移除，其中队列的头部是在队列中时间最长的元素，队列的尾部是在队列中时间最短的元素。在LinkedBlockingQueue类中分别用2个不同的锁takeLock、putLock来保护队头和队尾操作。如下图所示：
 
 ![image](http://img.blog.csdn.net/20160808012153461?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
 
# 1.2 类图

![image](http://img.blog.csdn.net/20160808004715770?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#1.3 源码分析

## 1.3.1 属性与链表节点类


```
//链表节点类，next指向下一个节点。如果下一个节点时null表示没有节点了。  
static class Node<E> {  
    E item;  
  
    Node<E> next;  
  
    Node(E x) { item = x; }  
}  
  
// 最大容量上限，默认是 Integer.MAX_VALUE  
private final int capacity;  
  
// 当前元素数量，这是个原子类。  
private final AtomicInteger count = new AtomicInteger(0);  
  
// 头结点  
private transient Node<E> head;  
  
// 尾结点  
private transient Node<E> last;  
  
// 队头访问锁  
private final ReentrantLock takeLock = new ReentrantLock();  
  
// 队头访问等待条件、队列  
private final Condition notEmpty = takeLock.newCondition();  
  
// 队尾访问锁  
private final ReentrantLock putLock = new ReentrantLock();  
  
// 队尾访问等待条件、队列  
private final Condition notFull = putLock.newCondition();  
```

使用原子类AtomicInteger是因为读写分别使用了不同的锁，但都会访问这个属性来计算队列中元素的数量，所以它需要是线程安全的。关AtomicInteger详细请看我的这一篇文章：[【Java并发编程】深入分析AtomicInteger(二)](http://blog.csdn.net/liulongling/article/details/50547159)

## 1.3.2 offer操作


```
public boolean offer(E e) {  
    if (e == null) throw new NullPointerException();  
    final AtomicInteger count = this.count;  
    //当队列满时，直接返回了false，没有被阻塞等待元素插入  
    if (count.get() == capacity)  
        return false;  
    int c = -1;  
    Node<E> node = new Node(e);  
    //开启队尾保护锁  
    final ReentrantLock putLock = this.putLock;  
    putLock.lock();  
    try {  
        if (count.get() < capacity) {  
            enqueue(node);  
            //原则计数类  
            c = count.getAndIncrement();  
            if (c + 1 < capacity)  
                notFull.signal();  
        }  
    } finally {  
        //释放锁  
        putLock.unlock();  
    }  
    if (c == 0)  
        signalNotEmpty();  
    return c >= 0;  
}  
  
//在持有锁下指向下一个节点  
private void enqueue(Node<E> node) {  
    // assert putLock.isHeldByCurrentThread();  
    // assert last.next == null;  
    last = last.next = node;  
}  
```

## 1.3.3 put操作

```
//put 操作把指定元素添加到队尾，如果没有空间则一直等待。  
public void put(E e) throws InterruptedException {  
    if (e == null) throw new NullPointerException();  
    // Note: convention in all put/take/etc is to preset local var  
    // holding count negative to indicate failure unless set.  
    //注释：在所有的 put/take/etc等操作中变量c为负数表示失败，>=0表示成功。  
    int c = -1;  
    Node<E> node = new Node(e);  
    final ReentrantLock putLock = this.putLock;  
    final AtomicInteger count = this.count;  
    putLock.lockInterruptibly();  
    try {  
        /* 
         * Note that count is used in wait guard even though it is 
         * not protected by lock. This works because count can 
         * only decrease at this point (all other puts are shut 
         * out by lock), and we (or some other waiting put) are 
         * signalled if it ever changes from capacity. Similarly 
         * for all other uses of count in other wait guards. 
         */  
        /* 
         * 注意，count用于等待监视，即使它没有用锁保护。这个可行是因为 
         * count 只能在此刻（持有putLock）减小（其他put线程都被锁拒之门外）， 
         * 当count对capacity发生变化时，当前线程（或其他put等待线程）将被通知。 
         * 在其他等待监视的使用中也类似。 
         */  
        while (count.get() == capacity) {  
            notFull.await();  
        }  
        enqueue(node);  
        c = count.getAndIncrement();  
        // 还有可添加空间则唤醒put等待线程。  
        if (c + 1 < capacity)  
            notFull.signal();  
    } finally {  
        putLock.unlock();  
    }  
    if (c == 0)  
        signalNotEmpty();  
}  
```

## 1.3.4 take操作


```
//弹出队头元素，如果没有会被阻塞直到元素返回  
public E take() throws InterruptedException {  
    E x;  
    int c = -1;  
    final AtomicInteger count = this.count;  
  
    final ReentrantLock takeLock = this.takeLock;  
    takeLock.lockInterruptibly();  
    try {  
        while (count.get() == 0) {  
            notEmpty.await();//没有元素一直阻塞  
        }  
        x = dequeue();  
        c = count.getAndDecrement();  
        if (c > 1)//如果还有可获取元素，唤醒等待获取的线程。  
          notEmpty.signal();  
    } finally {  
        //拿到元素后释放锁  
        takeLock.unlock();  
    }  
    if (c == capacity)  
        signalNotFull();  
    return x;  
}  
  
//在持有锁下返回队列队头第一个节点  
private E dequeue() {  
    // assert takeLock.isHeldByCurrentThread();  
    // assert head.item == null;  
    Node<E> h = head;  
    Node<E> first = h.next;  
    h.next = h; // help GC  
    //出队后的节点作为头节点并将元素置空  
    head = first;  
    E x = first.item;  
    first.item = null;  
    return x;  
}  
```

## 1.3.5 remove操作
![image](http://img.blog.csdn.net/20160808114644732?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


```
//移除指定元素。  
public boolean remove(Object o) {  
    if (o == null) return false;  
    //对两把锁加锁  
    fullyLock();  
    try {  
        for (Node<E> trail = head, p = trail.next;  
                p != null;  
                trail = p, p = p.next) {  
            if (o.equals(p.item)) {  
                unlink(p, trail);  
                return true;  
            }  
        }  
        return false;  
    } finally {  
        fullyUnlock();  
    }  
}  
  
//p是移除元素所在节点，trail是移除元素的上一个节点  
void unlink(Node<E> p, Node<E> trail) {  
    // assert isFullyLocked();  
    // p.next is not changed, to allow iterators that are  
    // traversing p to maintain their weak-consistency guarantee.  
    p.item = null;  
    //将trail下一个节点指向p的下一个节点  
    trail.next = p.next;  
    if (last == p)  
        last = trail;  
    if (count.getAndDecrement() == capacity)  
        notFull.signal();  
}  
  
void fullyLock() {  
    putLock.lock();  
    takeLock.lock();  
}  
  
//释放锁时确保和加锁顺序一致  
void fullyUnlock() {  
    takeLock.unlock();  
    putLock.unlock();  
}  
```

注意，锁的释放顺序与加锁顺序是相反的。

> 作者：小毛驴，一个Java游戏服务器开发者　原文地址[：https://liulongling.github.io/](https://liulongling.github.io//)
