---
title: 并发之AQS
date: 2021-05-02 17:53:47
tags:
  - "并发"
  - "AQS"
categories: "并发"
---

## 并发之同步器(AbstractQueueSynchronize)

同步器是juc并发包中并发工具的基础,很多经常使用的同步工具类都是基于aqs实现的.例如ReentrantLock,CountDownLatch,Semphore等.

基本的实现方式是通过一个state变量控制资源是否已经被占用,以及一个CLH队列实现.

### 为什么要使用AQS,和synchronized相比有什么优势

<!--more-->

优势以下:

1. 可以非阻塞式的抢占锁,没有抢到直接返回失败,有效避免死锁发生的情况.
2. 可以控制阻塞等待的时间.
3. 可以能够响应线程中断,synchronized一旦进入阻塞等待状态,不可以取消无法被中断.
4. 也可以以共享占用资源的方式抢锁,可以控制线程数.

### JMM happens-before先行发生规则

如果想要保证两个操作的先后顺序是可控的,必须满足happens-before规则. 否则jvm可以随意进行重排序.

规则:

- 监视器锁规则: 对同一个锁的解锁必须先于加锁.
- volatile规则: 对volatile变量的写入操作必须在读取操作之前执行.
- 线程启动规则: Thread.start()操作必须在线程内任何操作之前执行.
- 线程结束规则: 线程中的任何操作必须在其他线程检测到该线程已经结束之前执行.
- 中断规则: 当一个线程调用另一个线程的interrupt时必须在被中断线程检测到interrupt调用之前执行.
- 终结器规则: 对象的构造函数必须在启动对象的终结器之前执行完成
- 传递性: 如果a在b之前,并且b在c之前,那么a必须在c之前执行.

### AQS类

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
        //标识资源状态,为0标识未被占用. >0标识已被占用.
        private volatile int state;
        /**
         * 线程没有抢占到锁. 被封装成队列的一个虚拟节点,进入阻塞状态,通过此类实现.
         *
         * 通过Node可以实现两个队列,第一个是通过prev和next实现等待资源的双向队列.第二个是通过nextWaiter实现等待条件的队列.
         */
        static final class Node{
            //共享模式占有锁
            static final Node SHARED = new Node();
            //独占模式占有锁
            static final Node EXCLUSIVE = null;
            //等待队列中节点的状态. 只有四种状态
            //0. 初始化时默认值.
            //1. 标识线程已被取消
            //-1. 标识后续节点需要被取消阻塞
            //-2  标识节点在等待条件. waiting on condition
            //-3 表示有资源可用，新head结点需要继续唤醒后继结点（共享模式下，多线程并发释放资源，而head唤醒其后继结点后，需要把多出来的资源留给后面的结点；设置新的head结点时，会继续唤醒其后继结点）
            volatile int waitStatus;
            volatile Node prev; // 前驱结点
            volatile Node next; // 后继结点
            volatile Thread thread; // 结点对应的线程
            //等待队列中下一个等待条件的节点
            Node nextWaiter;
        }
        //... 
    }
```

#### 获取资源 `acquire(int)` 方法.

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
            selfInterrupt();
        }
    }
```

1. 首先调用`tryAcquire(int)` 尝试获取资源,这个方法的具体实现由子类定义,aqs只提供了模板.
2. 如果获取失败,调用`addWaiter(mode)`把当前线程封装为一个node对象添加到等待队列队尾. mode代表抢占资源的模式.

```java
private Node addWaiter(Node mode) {
    //创建一个新的节点
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    //获取到队尾的节点
    Node pred = tail;
    if (pred != null) {
        //把新节点的前驱节点指向队尾的节点
        node.prev = pred;
        //使用cas设置队尾节点为新节点
        if (compareAndSetTail(pred, node)) {
            //把之前队尾节点的后继节点指向新节点
            pred.next = node;
            return node;
        }
    }
    //如果等待队列为空或者cas添加队尾失败,则调用enq()添加
    enq(node);
    return node;
}
//cas自旋入队列
private Node enq(final Node node) {
    for (;;) {
        //获取尾节点
        Node t = tail;
        //如果队列为空,初始化一个空节点当作队头和队尾
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //加入到队尾中
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

3. 最后调用`acquireQueued(Node,arg)`挂起当前线程.

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取前驱节点
            final Node p = node.predecessor();
            //如果node的前驱节点是头节点,则node节点可以尝试抢占资源
            if (p == head && tryAcquire(arg)) {
                //如果抢占成功,把当前节点设为头节点. 所以head指向的节点就是正在占用资源的节点或者null.
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

#### 释放资源 `release(int)` 方法.

```java
public final boolean release(int arg) {
    //尝试释放,由子类实现.
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0) {
            //唤醒头节点
            unparkSuccessor(h);
        }
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    //如果状态是负数,把waitStatus置为0
    int ws = node.waitStatus;
    if (ws < 0) {
        compareAndSetWaitStatus(node, ws, 0);
    }

    //获取头节点的后继节点
    Node s = node.next;
    //如果为空或者取消,寻找下一个可用节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从队尾向前寻找未被取消的等待节点
        for (Node t = tail; t != null && t != node; t = t.prev) {
            if (t.waitStatus <= 0) {
                s = t;
            }
        }
    }
    //唤醒
    if (s != null) {
        LockSupport.unpark(s.thread);
    }
}

```

### 参考资料

- [AQS](https://redspider.gitbook.io/concurrent/di-er-pian-yuan-li-pian/11)
- Java并发编程实战