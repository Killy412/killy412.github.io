---
title: Netty-NIO基础
date: 2021-06-27 22:25:47
tags:
  - "Netty"
  - "NIO"
categories:
  - "Netty"
---

## IO基础知识

在1.4版本之前，Java IO类库是阻塞IO；从1.4版本开始，引进了新的异步IO库，被称为JavaNew IO类库，简称为JAVA NIO.New IO类库的目标，就是要让Java支持非阻塞IO，基于这个原因，更多的人喜欢称Java NIO为非阻塞IO（Non-Block IO），称“老的”阻塞式Java IO为OIO（Old IO）。总体上说，NIO弥补了原来面向流的OIO同步阻塞的不足，它为标准Java代码提供了高速的、面向缓冲区的IO。

<!--more-->

### IO模型

1. 同步阻塞(**Blocking IO**)
2. 同步非阻塞(**Non-blocking IO**)
3. IO多路复用(**IO Multiplexing**):经典的Reactor反应器设计模式,也被称为异步阻塞IO
4. 异步IO(**Asynchronous IO**)

> 阻塞和非阻塞:**阻塞和非阻塞描述的是调用方的状态**.
> **阻塞**:在得到结果之前,一直在等待.**非阻塞**:如果没有得到结果就返回,等一会儿再去请求,直到返回为止.
> 阻塞IO指的是需要内核IO操作彻底完成之后,才返回到用户空间执行用户的操作.阻塞指的是用户空间程序的执行状态.
>
> 同步和异步: **描述的是结果的发出**
> 同步IO是指用户空间的线程主动发起IO请求的一方,内核空间是被动接收方.异步IO反过来,系统内核是主动发起IO请求的一方.
> **同步** :在没获得结果之前不会返回给调用方,如果调用方是阻塞的,就会一直阻塞.如果调用方是非阻塞的,就会先回去,等会儿再来
> **异步**: 调用方一来,会直接返回.等执行完逻辑,通过回调函数返回给调用方
>
> 个人理解:
>
> 阻塞/非阻塞是指调用方在调用时的状态. 描述的是调用方发起某个请求之后,到请求有响应结果这段时间内的调用方的状态.如果调用方在一直等待,是阻塞. 如果可以继续做别的事情,是非阻塞.
>
> 同步/异步指对调用方式的描述. 调用某个请求,如果需要等待,则是同步. 不需要等待,是异步. 同步异步是从行为角度描述问题.

#### 小结

Netty是建立在NIO基础之上,在NIO之上又提供了更高层次的抽象
Accept连接可以使用单独的线程池去处理,读写操作又是另外的线程池处理

### 基础

> 关于Netty：
>
> Netty是一个高性能、异步事件驱动的NIO框架，它提供了对TCP、UDP和文件传输的支持，作为一个异步NIO框架，Netty的所有IO操作都是异步非阻塞的，通过Future-Listener机制，用户可以方便的主动获取或者通过通知机制获得IO操作结果。作为当前最流行的NIO框架，Netty在互联网领域、大数据分布式计算领域、游戏行业、通信行业等获得了广泛的应用，一些业界著名的开源组件也基于Netty的NIO框架构建。
> 
> netty是对Java NIO和Java线程池技术的封装

- NIO是异步非阻塞
  使用事件机制,用一个线程把accept/读写操作/请求处理的逻辑全干了,没有任务会把线程休眠,直到下一个事件来到.这样的线程成为NIO线程

#### NIO组件

非阻塞型IO(non-blocking io),包含三个核心概念 Selector,Channel,Buffer

NIO以下三个核心组件

##### Channel 通道

所有NIO操作都来自Channel,Channel是数据读取或者写入的目的地.一个连接就是用一个Channel通道表示.最重要的四种Channel实现	
- FileChannel 文件通道,用于文件的数据读写.
- SocketChannel 套接字通道(网络通道),用于Socket套接字TCP连接的数据读写.(连接传输)
- ServerSocketChannel 服务器监听通道.允许我们监听TCP连接请求.(连接监听)
- DatagramChannel数据报通道，用于UDP协议的数据读写.

##### Buffer  缓冲区

本质是一个缓冲区,可以进行读写数据.是一个抽象类,内部是一个内存块(数组).
- capacity 容量,可缓冲对象的数量
- position 位置,读写的位置
- limit 读写的最大上限
	

	在flip翻转时，属性的调整，将涉及position、limit两个属性，这种调整比较微妙，不是太好理解，举一个简单例子：首先，创建缓冲区。刚开始，缓冲区处于写模式。position为0, limit为最大容量。然后，向缓冲区写数据。每写入一个数据，position向后面移动一个位置，也就是position的值加1。假定写入了5个数，当写入完成后，position的值为5。这时，使用（即调用）flip方法，将缓冲区切换到读模式。limit的值，先会被设置成写模式时的position值。这里新的limit是5，表示可以读取的最大上限是5个数。同时，新的position会被重置为0，表示可以从0开始读

> 总体来说，使用Java NIO Buffer类的基本步骤如下：
> 1. 使用创建子类实例对象的allocate()方法，创建一个Buffer类的实例对象。
> 2. 调用put方法，将数据写入到缓冲区中。
> 3. 写入完成后，在开始读取数据前，调用Buffer.flip()方法，将缓冲区转换为读模式。
> 4. 调用get方法，从缓冲区中读取数据。
> 5. 读取完成后，调用Buffer.clear() 或Buffer.compact()方法，将缓冲区转换为写入模式。

##### Selector 选择器/多路复用器

IO多路复用,一个线程可以监视多个文件描述符(一个网络连接,操作系统底层使用文件描述符表示),一旦其中的一个/多个文件描述符可读/可写,系统内核就通知该进程/线程.在Java应用层面,使用选择器实现对文件描述符的监视.
选择器是一个IO事件的查询器,通过选择器,线程可以查询多个通道IO事件的状态.

**使用一个Selector来管理多个Channel,可以是SocketChannel,或者是ServerSocketChannel,将各个通道注册到指定的Selector上,指定监听的事件. 然后使用一个线程轮询Selector,去检查是否有准备好的事件,当通道可读或可写,才去真正开始读写,这样就不用给每个channel开启一个线程,大大提高线程可用.**

##### Selector的底层实现

Selector是对底层操作系统实现的一个抽象,管理通道状态都是底层操作系统实现的.

- **select: 20世纪80年代出现的.只支持注册1024个socket.**
- **poll: 1997年出现的. 最大的区别不再限制socket的数量**
- **epoll: 2002年随Linux内核2.5.44发布,可以直接返回可用的channel,事件复杂度是O(1);**

**select和poll都有一个问题,只知道有多少通道准备好,不知道具体哪几个channel,扫描一遍的事件复杂度是O(n),对应epoll是O(1).**

#### NIO工作流程

1. 首先创建一个Selector,用来监视管理各个Channel,也就是不同的客户端.相当于取代了以前BIO中的线程池,但是他只需要一个线程就可以处理多个Channel,没有上下文线程切换带来的消耗,提升了很大的性能
2. 创建一个ServerSocketChannel监听通信端口,并注册到Selector,让selector监听客户端的accept状态,也就是监听客户端请求
3. 客户端请求服务器,Selector就知道有客户端请求进来.然后可以得到客户端的SocketChannel,并为这个通道注册Read状态,也就是Selector会监听客户端发来的消息
4. 一旦接收到客户端的消息,就会用其他客户端的SocketChannel的Write状态,向他们转发客户端的消息

#### reactor线程模型

- reactor单线程模型: 所有操作都在一个线程中完成

- reactor多线程模型,在处理器链部分采用多线程,后端程序常用模型
  与单线程模型的最大区别是有一组NIO线程处理io操作. 有一个线程专门处理accept操作.
  
- reactor主从模型:多个acceptor的NIO线程用于接收客户端的连接
  将reactor分成两部分,manReactor负责处理新的socket连接,把已经建立连接的socket分发给subReactor.subReactor负责多路分离已连接的socket,读写网络数据/业务处理


#### AIO 异步IO

异步IO主要为了控制线程数量,减少过多线程带来的内存消耗和CPU在线程调度上的开销.

总共有三个类需要关注，分别是 **AsynchronousSocketChannel**，**AsynchronousServerSocketChannel** 和 **AsynchronousFileChannel**，只不过是在之前介绍的 FileChannel、SocketChannel 和 ServerSocketChannel 的类名上加了个前缀 **Asynchronous**。

Java异步IO提供了两种使用方式,分别是回调函数和Future实例.

1. 返回**Future**对象
2. 提供**CompletionHandler**回调函数

**AsynchronousChannelGroup**类. 异步IO一定存在一个线程池. 线程池负责处理任务,处理IO事件,回调等. 线程池就在group内部.group一旦关闭,线程池就会关闭.
