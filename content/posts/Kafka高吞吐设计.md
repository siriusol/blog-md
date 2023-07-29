---
title: "Kafka高吞吐设计"
subtitle: ""
date: 2023-07-29T22:43:16+08:00
draft: false
author: ""
authorLink: ""
authorEmail: ""
description: ""
keywords: ""
license: ""
comment: false
weight: 0

tags:
- 消息队列
- Kafka
categories:
- 消息队列

hiddenFromHomePage: false
hiddenFromSearch: false

summary: ""
resources:
- name: featured-image
  src: featured-image.jpg
- name: featured-image-preview
  src: featured-image-preview.jpg

toc:
  enable: true
math:
  enable: false
lightgallery: false
seo:
  images: []

# See details front matter: /theme-documentation-content/#front-matter
---

Kafka 采用了一系列的技术优化来保证高吞吐，这其中包括批量处理、压缩、零拷贝、磁盘顺序读写、页面缓存技术、Reactor 网络架构设计模式等。接下来主要从生产端、服务端和消费端三个方面来剖析和讨论。同时讨论一些高性能的设计方法，以及操作系统底层的工作模式，这些都有利于高效地设计出一个高吞吐的系统。

## 生产端

Kafka 高吞吐量的特性在生产端这里是怎么体现的呢？要想回答这个问题，首先得了解下生产端是如何发送消息的，这属于铺垫知识。下图详细描述了生产端发送消息的全部流程。

![](https://s0.lgstatic.com/i/image6/M00/51/01/CioPOWEDbJeAZ9G6AAhIbEVyoo0362.png)

<center>生产端发送消息流程图</center>

结合该图，我们可以看到发送一条消息需要经历 7 个步骤，这些步骤可以分为三大块，分别是 KafkaProducer 主线程、RecordAccumulator 缓存和 Sender 子线程。

* **KafkaProducer 主线程**，主要负责创建信息，并调用拦截器、序列化器、分区器分别对消息进行拦截、序列化和路由分区，然后对消息进行压缩，把压缩过的消息放入 RecordAccumulator 缓存中。

* **RecordAccumulator 缓存**，为每个分区创建了一个队列，这个队列是要发送到某个分区的消息集合。

* **Sender 子线程**，是真正发送消息的线程。满足一定条件时，KafkaProducer 主线程会激活 Sender 子线程。Sender 子线程从 RecordAccumulator 缓存中拿到要发送的消息，并把消息交给底层网络组件来发送。对于网络接收和网络发送的数据，网络组件会通过两个缓存集合来维护：completedReceives 是负责保存完成的网络接收的集合，completedSends 是负责保存完成的网络发送的集合。服务端成功响应返回给 Sender 子线程后，Sender 子线程就会删除 RecordAccumulator 缓存内已经发送成功的消息。

介绍完生产端的这个架构设计后，接下来就从以下三点解释一下这个架构从哪些方面提升了消息的吞吐量。

### 1. 多线程异步的设计

生产端在异步的设计上体现到了两个方面。

**第一个方面，KafkaProducer 主线程和 Sender 子线程各司其职，通过 RecordAccumulator 缓存交互数据。**

KafkaProducer 主线程有同步和异步两种发送方式，但是这两者底层的实现是相同的，都是通过 Sender 子线程异步发送消息实现的。不同的地方是同步场景下主线程会等待 Sender 子线程发送完消息再返回，而异步是不等待 Sender 子线程发送完消息就返回了。

KafkaProducer 主线程发送消息时并不是真正的网络发送，而是将消息放入 RecordAccumulator 中缓存，然后主线程就从 send() 方法返回，之后 KafkaProducer 主线程会不断调用 send() 方法把消息缓存到 RecordAccumulator 中，而不去在意消息是否发送出去了。真正发送消息的是 Sender 子线程，Sender 子线程从 RecordAccumulator 缓存中取出消息，然后调用底层网络组件完成消息的发送。

有的同学可能会有疑问：为什么不能把主线程和 Sender 子线程放到一个线程呢？一个线程里会有什么问题呢？

生产端发送消息有两个过程：创建消息和网络发送消息。这两个过程都有可能出现阻塞，比如，消息的创建依赖远程数据库或缓存，如果网络不好，线程就会阻塞在消息创建上；而生产端和服务端的通信不好时，也会导致出现阻塞的问题。如果这两个过程放到一个线程里的话，那么其中有一个发送阻塞，就会影响另一个过程的执行。

但是如果我们把创建消息交给主线程负责，发送消息交给子线程负责，这样这两个过程相互不影响，同时有缓存作为缓冲，很好地起到“削峰填谷”的作用。

第**二个方面，Sender 子线程和 Kafka 底层通信模块解耦。**

Sender 子线程最终是调用 Kafka 底层通信模块实现消息的发送和接收的。

我们知道 Java NIO 本质上是调用了 Linux 通信模块，Kafka 底层封装了 Java NIO 组件，特别是 org.apache.kafka.common.network.Selector（简称 KSelector）封装了 Java NIO 的 Selector 类，KSelector 在 Selector 的基础上增加了两个集合做缓冲，分别是 completedReceives 集合和 completedSends 集合，KSelector 发送成功和接收成功的消息都会放到这两个集合里。而 Sender 子线程通过 while(true) 循环不断地尝试从这两个集合获取消息，从而实现了这两个组件的解耦，道理也是一样，也是起到“削峰填谷”的作用，进而有利于高吞吐。

### 2. 在缓存中批量地获取数据，并做到高效的空间利用
这一点与 RecordAccumulator 类的设计关系很大，RecordAccumulator 类的设计图如下：

![](https://s0.lgstatic.com/i/image6/M00/50/F9/Cgp9HWEDbK6AMBwCAAPPZPimeXE261.png)

<center>RecordAccumulator 类设计图</center>

由图可以看到，在 RecordAccumulator 中有一个 CopyOnWriteMap 集合 batches。key 是主题分区，value 是 ProducerBatch 队列，每个分区都对应一个队列。队列中的元素是批次 ProducerBatch，消息就是封装在这些批次里进行缓存的。而消息发送的最小单位是 batch，也就是说一次消息发送可能不止一条消息，这样的设计大大减少了网络请求的次数，从而提升了网络读写的效率，进而提高了吞吐量。

接下来我们再来分析下消息的发送时机和逻辑。代码在 RecordAccumulator.drain() 方法内，其源码和源码注释如下：

```java
//五个判断条件决定是否是能发送的node
boolean sendable = full || expired || exhausted || closed || flushInProgress();
//能发送且没有正在退避
if (sendable && !backingOff) {
    //如果是能发送就加入readyNodes集合
    readyNodes.add(leader);
} else {
    long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
    //还剩多久：需要等待的时间-已经等待的时间
    nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
}
```


这里我重点解释下决定是否发送的布尔型变量 sendable 的判断逻辑：五个判断条件只要有一个满足就能发送消息。这五个条件可总结为如下。

* full：deque 是否大于 1，或 deque 的第一个 ProducerBatch 是否满了。

* expired：ProducerBatch 在 deque 里是否超时。

* exhausted：BufferPool 是否正在释放空间。

* closed：生产者是否准备正常关闭了。

* flushInProgress：是否在 flush 操作，这个 flush 是把暂存消息立即发送的标记。

第一个判断条件是 deque 是否大于 1，或 deque 的第一个 ProducerBatch 是否满了，在 Broker 负载没满的情况下，deque 的第一个 ProducerBatch 是否满了是大部分情况下发送消息的时机。所以说，生产者发送消息并不是一条条发送的，而是一个一个 batch 发送的。

接下来我们再分析下**生产端高效的空间利用特性**。

缓存的空间分配是由 BufferPool 组件完成的，下面是其工作原理图：

![](https://s0.lgstatic.com/i/image6/M00/51/02/CioPOWEDbMmAcpiVAALwOMYqWmQ225.png)

<center>BufferPool 组件工作原理图</center>

整个 BufferPool 的大小默认为 32M，内部内存区域分为两块：固定大小内存块集合 free 和非池化缓存 nonPooledAvailableMemory。固定大小内存块默认大小为 16K。当 ProducerBatch 向 BufferPool 申请一个大小为 size 的内存块时，BufferPool 会根据 size 的大小判断由哪个内存区域分配内存块。

当 ProducerBatch 的数据发送成功后，ProducerBatch 并不会销毁，而是继续留在集合 free 中，这样需要 ProducerBatch 的时候就直接从集合中拿出，就不用频繁地销毁和重建了。其实 ProducerBatch 的底层是 Java NIO ByteBuffer，ByteBuffer 的创建和销毁是很消耗 CPU 资源的，这样的设计实现了 ByteBuffer 的重用，从而大大减少了对资源的消耗。

### 3. 消息的压缩
消息压缩是在业务主线程 KafkaProducer 完成的，消息的压缩大大减少了本地内存、网络通信和服务端存储的压力。

目前主要有 4 种压缩算法，分别是 gzip、snappy、lz4 和 zstd。你可以根据生产环境实际情况来配置适合自己的压缩算法，评估一个压缩算法一般是从压缩解压的速度和压缩率两方面去权衡的。当机器 CPU 配置比较高而带宽比较低的时候，可以考虑压缩率高而压缩速度低的算法；相反，当机器 CPU 配置比较低而带宽比较高的时候，可以考虑压缩率低而解压速度比较高的算法。

Kafka 生产端对于高吞吐的设计我就介绍到这里，接下来我继续介绍 Kafka 服务端针对高吞吐的设计特点。

## 服务端
服务端针对高吞吐有几个设计特点：网络层的 Reactor 设计模式、顺序写、页缓存和零拷贝。接下来我会按照顺序分别为你详细讲解。

### 1. 网络层的 Reactor 设计模式
网络层的设计图如下：

![](https://s0.lgstatic.com/i/image6/M00/51/02/CioPOWEDbNqARKWBAAJy6zlp8aI724.png)

<center>服务端网络架构设计图</center>

这里我就结合该图解释一下网络层的架构设计。

整个服务端的网络架构分为 4 个层次：①Acceptor 线程构成的**连接创建层**，负责创建和客户端的连接；②Processor 线程类构成的**网络事件处理层**；③由 RequestChannel 构成的请求和响应的**缓冲层**；④由 KafkaRequestHandler 和 KafkaApis 构成的真正的**业务处理层**。

这样的设计有什么优势呢？

第一，我们先思考为什么要把 Acceptor 线程和 Processor 线程分开。如果不分开，网络读写的量很大势必造成大量线程阻塞，导致服务端对 OP_ACCEPT 事件响应不及时，进而连接失败。同时，如果服务端刚启动瞬时来了很多连接，大量的线程都去建立新的连接了，那么网络读写事件的处理就会慢下来，也会引起读写超时等问题。

Acceptor 线程和 Processor 线程分为两层这样的设计让连接的创建和网络读写事件的处理分开，同时还可以配置 Processor 线程的数量，这样做不会被极端场景影响到整体的响应时间，同时也是符合 Reactor 设计模式的。（Reactor 模式又被称为反应器模式或应答者模式，是基于事件驱动的设计模式，如有需要你可以查阅相关的资料来学习，这里就不过多赘述了。）

第二，Processor 线程解析完请求后并不是直接交给业务线程处理，而是放到 RequestChannel 的请求队列里，这样做避免在高并发场景下业务线程（即调用底层组件的线程）工作过于饱和而造成超时的情况出现。

第三，KafkaRequestHandlerPool 线程池先消费 RequestChannel 类里的请求队列，然后通过调用 KafkaApis 实现对底层组件的调用。这样做既可以实现网络处理和调用底层组件的解耦，也可以根据实际请求，随时调整 KafkaRequestHandlerPool 线程池的线程数，调整调用底层组件的能力。

第四，KafkaApis 类会把响应放入对应的 Processor 线程里的响应集合里，而不是直接让 Processor 把响应发送给客户端，这样做实现了业务线程和网络操作线程的解耦，避免了高并发时线程工作过于饱和而造成的延迟问题。

### 2. 顺序写
Kafka 写日志文件的时候用的是追加消息的形式，只在文件尾部顺序写消息，同时在文件头部顺序读取消息。消息队列不涉及修改消息，所以不需要随机写。这样的设计即使用的是传统的磁盘，吞吐量也会很大。主要原因是操作系统对于顺序写和顺序读有优化，具体采用的是后写（对于写消息优化）和预读（对于读消息优化）。生产环境上经过测试，顺序写比随机写快 3 个数量级。

### 3. 页缓存
页缓存简单说就是把缓存当磁盘用，这样就避免了频繁地读写磁盘。

当一个进程要读取或写入磁盘文件的时候，系统会判断数据是否在内存中，如果在，就直接把内存中的数据返回给进程；如果不在，就读取磁盘文件，同时会多读一些连续的磁盘页放到内存中。这样下次再读取或写入时，系统会判断数据是否在内存中，只要是顺序地读写消息，命中率会很高的，大大减少了磁盘访问的次数，提高了服务端的吞吐量。

### 4. 零拷贝
这里我们以消费者读取消息为例，服务端要从磁盘拷贝数据然后网络发送，如果不采用零拷贝的话，会发生什么样的事情呢？如下图所示：

![](https://s0.lgstatic.com/i/image6/M00/51/02/CioPOWEDbOuAD4z_AAE6h6G4b-o112.png)

<center>非零拷贝过程图</center>

首先，应用程序调用 read() 方法时需要从用户态切换到内核态，将数据从磁盘上取出来保存到内核缓冲区中；然后，内核缓冲区中的数据传输到应用程序，此时 read() 方法调用结束，从内核态切换到用户态。之后，应用程序执行 send() 方法，需要从用户态切换到内核态，将数据传输给 Socket Buffer；最后，内核会将 Socket Buffer 中的数据发送到网卡，再发送到远程节点，此时 send() 方法结束，从内核态切换到用户态。

可以看到，这个过程共涉及四次 CPU 上下文切换和四次数据复制，并且有两次复制操作是由 CPU 完成的，另外两次由 DMA 完成。在这个过程中，数据本身没有任何修改，仅仅是从磁盘复制到了网卡缓冲区中，于是会浪费大量的 CPU 周期。

那采用零拷贝又会发生什么呢？如下过程图：

![](https://s0.lgstatic.com/i/image6/M00/50/F9/Cgp9HWEDbPiAUHBIAAEMXenJgF4404.png)

<center>零拷贝过程图</center>

首先，应用程序调用 transferTo() 方法，从用户态转换为内核态，DMA 会将文件数据发送到内核缓冲区；然后，Socket Buffer 追加数据的描述信息；最后，DMA 将内核缓冲区的数据发送到网卡缓冲区，这样就完全解放了 CPU，实现了零拷贝。

也就是说，所谓“零拷贝”是 CPU 不参与拷贝数据的工作，可以节省大量的 CPU 周期，同时减少了两次 CPU 在用户态和内核态的切换。这样大大减少了 CPU 的负载，从而提升了吞吐量。

## 消费端
相较生产端和服务端，消费端提升吞吐量的策略就没那么复杂了。

一般来讲，消费端提升吞吐量的方式主要就是通过解耦，消费者在消费消息的时候，也是有两个线程分别来拉取消息任务线程和网络 IO 任务线程。下图描述了拉取消息的过程：

![](https://s0.lgstatic.com/i/image6/M00/50/F9/Cgp9HWEDbQeAGhdjAAsVGWUzvKw810.png)

<center>消费者拉取消息的过程</center>

通过该图可以看到，消费者拉取完消息后并不是直接处理，而是放到一个缓存里，等待其他任务处理。

那消费者为什么不直接从 Broker 拉取消息，而是先把消息拉取过来放入缓存再等着获取呢？可以看下面的关系图：

![](https://s0.lgstatic.com/i/image6/M00/51/02/CioPOWEDbRmAbW47AAxkOLj86Is054.png)

<center>拉取消息任务和网络 IO 任务的关系图</center>

如图所示，拉取消息任务和网络 IO 任务是解耦的，网络 IO 任务会事先把消息拉取到消费者缓存里，然后等待拉取消息任务读取缓存里的消息。这样做的好处是拉取消息任务拉取消息的时候不会造成 IO 阻塞，可以提高拉取消息任务的效率，并最终提升整体的吞吐量。

## 总结
今天我主要从生产端、服务端和消费端三个方面给你介绍了 Kafka 为提高吞吐量而做的一些设计。

* 生产端主要是通过消息压缩、消息缓存批量发送、异步解耦等方面提升吞吐量的。

* 服务端采用的优化技术比较多，比如，网络层的 Reactor 设计提升了网络层的吞吐，顺序写、页缓存、零拷贝这些是利用操作系统的优化点来实现存储层读写的吞吐量。

* 消费端主要是通过线程异步解耦的方式提升了拉取消息的效率，进而提升消费者的吞吐量。

结合我多年的工作经验来看，搞清楚 Kafka 对于高吞吐的设计思路是有很多好处的。

首先，了解这个设计原理后能更好地调优 Kafka 的性能，比如服务端网络层有 Acceptor 和 Processor 两种线程，当网络读写比较频繁的时候，你可以通过增加 Processor 线程数来提升网络吞吐。

另外，这些设计原理对你设计其他系统的时候也有很大的借鉴意义。比如，你在设计高吞吐系统的时候，完全可以借鉴生产者用不同的线程完成不同的任务，实现任务的解耦，防止某个任务延迟造成整体变慢。还有，利用操作系统本身的特性优化吞吐量也是值得学习的，比如，页缓存、顺序读写、零拷贝等，大大提升了系统的吞吐量。在工作中，你都可以好好利用这些优秀的设计来实现高吞吐、高性能的系统。
