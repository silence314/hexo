---
title: Netty学习笔记
tags:
  - Netty
  - IO
date: 2020-09-21 10:33:40
description: Spring MVC项目升级Spring Boot过程中有许多jar包冲突问题，其中不少是Netty的冲突报错，决定了解一下Netty是个啥
cover: /blogImg/netty服务端创建时序图.png
categories: Netty
typora-root-url: ../../themes/butterfly/source
---

了解Netty的过程中，看着各种网络编程，套接字，仿佛想到了大学时候网络编程那门课，一样头秃，这先大概总结一下netty的基础知识，对他有一个大概的了解，具体更深的源码以及实现原理后面学习明白了再进行总结。

# 首先，IO/NIO/BIO/AIO都是个啥

## 同步与异步

- **同步** ：两个同步任务相互依赖，并且一个任务必须以依赖于另一任务的某种方式执行。 比如在`A->B`事件模型中，你需要先完成 A 才能执行B。 再换句话说，同步调用中被调用者未处理完请求之前，调用不返回，调用者会一直等待结果的返回。
- **异步**： 两个异步的任务完全独立的，一方的执行不需要等待另外一方的执行。再换句话说，异步调用种一调用就返回结果不需要等待结果返回，当结果返回的时候通过回调函数或者其他方式拿着结果再做相关事情，

## 阻塞与非阻塞

- **阻塞：** 阻塞就是发起一个请求，调用者一直等待请求结果返回，也就是当前线程会被挂起，无法从事其他任务，只有当条件就绪才能继续。
- **非阻塞：** 非阻塞就是发起一个请求，调用者不用一直等着结果返回，可以先去干其他事情。

![同步异步:阻塞非阻塞](/blogImg/同步异步:阻塞非阻塞.png)

![异步非阻塞](/blogImg/异步非阻塞.png)

**IO模型主要分类：**

- 同步(synchronous) IO和异步(asynchronous) IO
- 阻塞(blocking) IO和非阻塞(non-blocking)IO
- 同步阻塞(blocking-IO)简称BIO
- 同步非阻塞(non-blocking-IO)简称NIO
- 异步非阻塞(synchronous-non-blocking-IO)简称AIO

## IO

IO的全称是：Input/Output

## BIO

BIO全称是Blocking IO，是JDK1.4之前的传统IO模型，本身是同步阻塞模式。 线程发起IO请求后，一直阻塞IO，直到缓冲区数据就绪后，再进入下一步操作。针对网络通信都是一请求一应答的方式，虽然简化了上层的应用开发，但在性能和可靠性方面存在着巨大瓶颈，试想一下如果每个请求都需要新建一个线程来专门处理，那么在高并发的场景下，机器资源很快就会被耗尽。

BIO 就是传统的 [java.io](http://java.io/) 包，它是基于流模型实现的，交互的方式是同步、阻塞方式，也就是说在读入输入流或者输出流时，在读写动作完成之前，线程会一直阻塞在那里，它们之间的调用时可靠的线性顺序。它的有点就是代码比较简单、直观；缺点就是 IO 的效率和扩展性很低，容易成为应用性能瓶颈

## NIO

NIO也叫Non-Blocking IO 是同步非阻塞的IO模型。线程发起io请求后，立即返回（非阻塞io）。同步指的是必须等待IO缓冲区内的数据就绪，而非阻塞指的是，用户线程不原地等待IO缓冲区，可以先做一些其他操作，但是要定时轮询检查IO缓冲区数据是否就绪。Java中的NIO 是new IO的意思。其实是NIO加上IO多路复用技术。普通的NIO是线程轮询查看一个IO缓冲区是否就绪，而Java中的new IO指的是线程轮询地去查看一堆IO缓冲区中哪些就绪，这是一种IO多路复用的思想。IO多路复用模型中，将检查IO数据是否就绪的任务，交给系统级别的select或epoll模型，由系统进行监控，减轻用户线程负担。

NIO主要有buffer、channel、selector三种技术的整合，通过零拷贝的buffer取得数据，每一个客户端通过channel在selector（多路复用器）上进行注册。服务端不断轮询channel来获取客户端的信息。channel上有connect,accept（阻塞）、read（可读）、write(可写)四种状态标识。根据标识来进行后续操作。所以一个服务端可接收无限多的channel。不需要新开一个线程。大大提升了性能。

NIO 是 Java 1.4 引入的 java.nio 包，提供了 Channel、Selector、Buffer 等新的抽象，可以构建多路复用的、同步非阻塞 IO 程序，同时提供了更接近操作系统底层高性能的数据操作方式。

## AIO

AIO是真正意义上的异步非阻塞IO模型。 上述NIO实现中，需要用户线程定时轮询，去检查IO缓冲区数据是否就绪，占用应用程序线程资源，其实轮询相当于还是阻塞的，并非真正解放当前线程，因为它还是需要去查询哪些IO就绪。而真正的理想的异步非阻塞IO应该让内核系统完成，用户线程只需要告诉内核，当缓冲区就绪后，通知我或者执行我交给你的回调函数。

AIO可以做到真正的异步的操作，但实现起来比较复杂，支持纯异步IO的操作系统非常少，目前也就windows是IOCP技术实现了，而在Linux上，底层还是是使用的epoll实现的。

AIO 是 Java 1.7 之后引入的包，是 NIO 的升级版本，提供了异步非堵塞的 IO 操作方式，所以人们叫它 AIO（Asynchronous IO），异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

# 然后，epoll/poll/select又是啥

select，poll，epoll都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

在介绍select、poll、epoll前，有必要说说linux(2.6+)内核的事件wakeup callback机制，这是IO多路复用机制存在的本质。Linux通过socket睡眠队列来管理所有等待socket的某个事件的process，同时通过wakeup机制来异步唤醒整个睡眠队列上等待事件的process，通知process相关事件发生。通常情况，socket的事件发生的时候，其会顺序遍历socket睡眠队列上的每个process节点，调用每个process节点挂载的callback函数。在遍历的过程中，如果遇到某个节点是排他的，那么就终止遍历，总体上会涉及两大逻辑：（1）睡眠等待逻辑；（2）唤醒逻辑。

1. 睡眠等待逻辑：涉及select、poll、epoll_wait的阻塞等待逻辑

> [1]select、poll、epoll_wait陷入内核，判断监控的socket是否有关心的事件发生了，如果没，则为当前process构建一个wait_entry节点，然后插入到监控socket的sleep_list
> [2]进入循环的schedule直到关心的事件发生了
> [3]关心的事件发生后，将当前process的wait_entry节点从socket的sleep_list中删除。


2. 唤醒逻辑：

> [1]socket的事件发生了，然后socket顺序遍历其睡眠队列，依次调用每个wait_entry节点的callback函数
> [2]直到完成队列的遍历或遇到某个wait_entry节点是排他的才停止。
> [3]一般情况下callback包含两个逻辑：1.wait_entry自定义的私有逻辑；2.唤醒的公共逻辑，主要用于将该wait_entry的process放入CPU的就绪队列，让CPU随后可以调度其执行。


下面就上面的两大逻辑，分别阐述select、poll、epoll的异同，为什么epoll能够比select、poll高效。

## select

在一个高性能的网络服务上，大多情况下一个服务进程(线程)process需要同时处理多个socket，我们需要公平对待所有socket，对于read而言， 哪个socket有数据可读，process就去读取该socket的数据来处理。于是对于read，一个朴素的需求就是关心的N个socket是否有数据”可读”，也就是我们期待”可读”事件的通知，而不是盲目地对每个socket调用recv/recvfrom来尝试接收数据。我们应该block在等待事件的发生上，这个事件简单点就是”关心的N个socket中一个或多个socket有数据可读了”，当block解除的时候，就意味着，我们一定可以找到一个或多个socket上有可读的数据。另一方面，根据上面的socket wakeup callback机制，我们不知道什么时候，哪个socket会有读事件发生，于是，process需要同时插入到这N个socket的sleep_list上等待任意一个socket可读事件发生而被唤醒，当时process被唤醒的时候，其callback里面应该有个逻辑去检查具体那些socket可读了。

于是，select的多路复用逻辑就清晰了，select为每个socket引入一个poll逻辑，该poll逻辑用于收集socket发生的事件，对于可读事件来说，简单伪码如下：

```c
poll()
{
    //其他逻辑
    if (recieve queque is not empty)
    {
        sk_event |= POLL_IN；
    }
   //其他逻辑
}
```

接下来就到select的逻辑了，下面是select的函数原型：5个参数，后面4个参数都是in/out类型(值可能会被修改返回)

```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

当用户process调用select的时候，select会将需要监控的readfds集合拷贝到内核空间（假设监控的仅仅是socket可读），然后遍历自己监控的socket sk，挨个调用sk的poll逻辑以便检查该sk是否有可读事件，遍历完所有的sk后，如果没有任何一个sk可读，那么select会调用schedule_timeout进入schedule循环，使得process进入睡眠。如果在timeout时间内某个sk上有数据可读了，或者等待timeout了，则调用select的process会被唤醒，接下来select就是遍历监控的sk集合，挨个收集可读事件并返回给用户了，相应的伪码如下：

```c
for (sk in readfds)
{
    sk_event.evt = sk.poll();
    sk_event.sk = sk;
    ret_event_for_process;
}
```

到这里，我们有三个问题需要解决：

1. 被监控的fds集合限制为1024，1024太小了，我们希望能够有个比较大的可监控fds集合 
2. fds集合需要从用户空间拷贝到内核空间的问题，我们希望不需要拷贝 
3. 当被监控的fds中某些有数据可读的时候，我们希望通知更加精细一点，就是我们希望能够从通知中得到有可读事件的fds列表，而不是需要遍历整个fds来收集。

## poll

poll和select非常相似，poll并没着手解决性能问题，poll只是解决了select的问题(1)fds集合大小1024限制问题。下面是poll的函数原型，poll改变了fds集合的描述方式，使用了pollfd结构而不是select的fd_set结构，使得poll支持的fds集合限制远大于select的1024。poll虽然解决了fds集合大小1024的限制问题，但是，它并没改变大量描述符数组被整体复制于用户态和内核态的地址空间之间，以及个别描述符就绪触发整体描述符集合的遍历的低效问题。poll随着监控的socket集合的增加性能线性下降，poll不适合用于大并发场景。

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

## epoll

在计算机行业中，有两种解决问题的思想：

1. 计算机科学领域的任何问题, 都可以通过添加一个中间层来解决
2. 变集中(中央)处理为分散(分布式)处理

### fds集合拷贝问题的解决

对于IO多路复用，有两件事是必须要做的(对于监控可读事件而言)：1. 准备好需要监控的fds集合；2. 探测并返回fds集合中哪些fd可读了。细看select或poll的函数原型，我们会发现，每次调用select或poll都在重复地准备(集中处理)整个需要监控的fds集合。然而对于频繁调用的select或poll而言，fds集合的变化频率要低得多，我们没必要每次都重新准备(集中处理)整个fds集合。

于是，epoll引入了epoll_ctl系统调用，将高频调用的epoll_wait和低频的epoll_ctl隔离开。同时，epoll_ctl通过(EPOLL_CTL_ADD、EPOLL_CTL_MOD、EPOLL_CTL_DEL)三个操作来分散对需要监控的fds集合的修改，做到了有变化才变更，将select或poll高频、大块内存拷贝(集中处理)变成epoll_ctl的低频、小块内存的拷贝(分散处理)，避免了大量的内存拷贝。同时，对于高频epoll_wait的可读就绪的fd集合返回的拷贝问题，epoll通过内核与用户空间mmap(内存映射)同一块内存来解决。mmap将用户空间的一块地址和内核空间的一块地址同时映射到相同的一块物理内存地址（不管是用户空间还是内核空间都是虚拟地址，最终要通过地址映射映射到物理地址），使得这块物理内存对内核和对用户均可见，减少用户态和内核态之间的数据交换。

另外，epoll通过epoll_ctl来对监控的fds集合来进行增、删、改，那么必须涉及到fd的快速查找问题，于是，一个低时间复杂度的增、删、改、查的数据结构来组织被监控的fds集合是必不可少的了。在linux 2.6.8之前的内核，epoll使用hash来组织fds集合，于是在创建epoll fd的时候，epoll需要初始化hash的大小。于是epoll_create(int size)有一个参数size，以便内核根据size的大小来分配hash的大小。在linux 2.6.8以后的内核中，epoll使用红黑树来组织监控的fds集合，于是epoll_create(int size)的参数size实际上已经没有意义了。

### 按需遍历就绪的fds集合

通过上面的socket的睡眠队列唤醒逻辑我们知道，socket唤醒睡眠在其睡眠队列的wait_entry(process)的时候会调用wait_entry的回调函数callback，并且，我们可以在callback中做任何事情。为了做到只遍历就绪的fd，我们需要有个地方来组织那些已经就绪的fd。为此，epoll引入了一个中间层，一个双向链表(ready_list)，一个单独的睡眠队列(single_epoll_wait_list)，并且，与select或poll不同的是，epoll的process不需要同时插入到多路复用的socket集合的所有睡眠队列中，相反process只是插入到中间层的epoll的单独睡眠队列中，process睡眠在epoll的单独队列上，等待事件的发生。同时，引入一个中间的wait_entry_sk，它与某个socket sk密切相关，wait_entry_sk睡眠在sk的睡眠队列上，其callback函数逻辑是将当前sk排入到epoll的ready_list中，并唤醒epoll的single_epoll_wait_list。而single_epoll_wait_list上睡眠的process的回调函数就明朗了：遍历ready_list上的所有sk，挨个调用sk的poll函数收集事件，然后唤醒process从epoll_wait返回。

整个过来可以分为以下几个逻辑：

（1）epoll_ctl EPOLL_CTL_ADD逻辑

> [1] 构建睡眠实体wait_entry_sk，将当前socket sk关联给wait_entry_sk，并设置wait_entry_sk的回调函数为epoll_callback_sk
> [2] 将wait_entry_sk排入当前socket sk的睡眠队列上


回调函数epoll_callback_sk的逻辑如下：

> [1] 将之前关联的sk排入epoll的ready_list
> [2] 然后唤醒epoll的单独睡眠队列single_epoll_wait_list


（2）epoll_wait逻辑

> [1] 构建睡眠实体wait_entry_proc，将当前process关联给wait_entry_proc，并设置回调函数为epoll_callback_proc
> [2] 判断epoll的ready_list是否为空，如果为空，则将wait_entry_proc排入epoll的single_epoll_wait_list中，随后进入schedule循环，这会导致调用epoll_wait的process睡眠。
> [3] wait_entry_proc被事件唤醒或超时醒来，wait_entry_proc将被从single_epoll_wait_list移除掉，然后wait_entry_proc执行回调函数epoll_callback_proc


回调函数epoll_callback_proc的逻辑如下：

> [1] 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件，对于监控可读事件而已，ready_list上的每个sk都是有数据可读的，这里的遍历必要的(不同于select/poll的遍历，它不管有没数据可读都需要遍历一些来判断，这样就做了很多无用功。)
> [2] 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process。


（3）epoll唤醒逻辑 整个epoll的协议栈唤醒逻辑如下(对于可读事件而言)：

> [1] 协议数据包到达网卡并被排入socket sk的接收队列
> [2] 睡眠在sk的睡眠队列wait_entry被唤醒，wait_entry_sk的回调函数epoll_callback_sk被执行
> [3] epoll_callback_sk将当前sk插入epoll的ready_list中
> [4] 唤醒睡眠在epoll的单独睡眠队列single_epoll_wait_list的wait_entry，wait_entry_proc被唤醒执行回调函数epoll_callback_proc
> [5] 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件
> [6] 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process。

epoll巧妙的引入一个中间层解决了大量监控socket的无效遍历问题。细心的同学会发现，epoll在中间层上为每个监控的socket准备了一个单独的回调函数epoll_callback_sk，而对于select/poll，所有的socket都公用一个相同的回调函数。正是这个单独的回调epoll_callback_sk使得每个socket都能单独处理自身，当自己就绪的时候将自身socket挂入epoll的ready_list。同时，epoll引入了一个睡眠队列single_epoll_wait_list，分割了两类睡眠等待。process不再睡眠在所有的socket的睡眠队列上，而是睡眠在epoll的睡眠队列上，在等待”任意一个socket可读就绪”事件。而中间wait_entry_sk则代替process睡眠在具体的socket上，当socket就绪的时候，它就可以处理自身了。

### ET(Edge Triggered 边沿触发) vs LT(Level Triggered 水平触发)

说到Epoll就不能不说说Epoll事件的两种模式了，下面是两个模式的基本概念

- Edge Triggered (ET) 边沿触发

.socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件

.socket的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时触发读事件

仅在缓冲区状态变化时触发事件，比如数据缓冲去从无到有的时候(不可读-可读)

- Level Triggered (LT) 水平触发

.socket接收缓冲区不为空，有数据可读，则读事件一直触发

.socket发送缓冲区不满可以继续写入数据，则写事件一直触发

符合思维习惯，epoll_wait返回的事件就是socket的状态

通常情况下，大家都认为ET模式更为高效，实际上是不是呢？下面我们来说说两种模式的本质：

我们来回顾一下，上面epoll唤醒逻辑 的第五个步骤

> [5] 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件


大家是不是有个疑问呢：挂在ready_list上的sk什么时候会被移除掉呢？其实，sk从ready_list移除的时机正是区分两种事件模式的本质。因为，通过上面的介绍，我们知道ready_list是否为空是epoll_wait是否返回的条件。于是，在两种事件模式下，步骤5如下：

对于Edge Triggered (ET) 边沿触发：

> [5] 遍历epoll的ready_list，将sk从ready_list中移除，然后调用该sk的poll逻辑收集发生的事件

对于Level Triggered (LT) 水平触发：

> [5.1] 遍历epoll的ready_list，将sk从ready_list中移除，然后调用该sk的poll逻辑收集发生的事件
> [5.2] 如果该sk的poll函数返回了关心的事件(对于可读事件来说，就是POLL_IN事件)，那么该sk被重新加入到epoll的ready_list中。


对于可读事件而言，在ET模式下，如果某个socket有新的数据到达，那么该sk就会被排入epoll的ready_list，从而epoll_wait就一定能收到可读事件的通知(调用sk的poll逻辑一定能收集到可读事件)。于是，我们通常理解的缓冲区状态变化(从无到有)的理解是不准确的，准确的理解应该是是否有新的数据达到缓冲区。

而在LT模式下，某个sk被探测到有数据可读，那么该sk会被重新加入到read_list，那么在该sk的数据被全部取走前，下次调用epoll_wait就一定能够收到该sk的可读事件(调用sk的poll逻辑一定能收集到可读事件)，从而epoll_wait就能返回。

**5.3.2 ET vs LT - 性能**

通过上面的概念介绍，我们知道对于可读事件而言，LT比ET多了两个操作：(1)对ready_list的遍历的时候，对于收集到可读事件的sk会重新放入ready_list；(2)下次epoll_wait的时候会再次遍历上次重新放入的sk，如果sk本身没有数据可读了，那么这次遍历就变得多余了。 在服务端有海量活跃socket的时候，LT模式下，epoll_wait返回的时候，会有海量的socket sk重新放入ready_list。如果，用户在第一次epoll_wait返回的时候，将有数据的socket都处理掉了，那么下次epoll_wait的时候，上次epoll_wait重新入ready_list的sk被再次遍历就有点多余，这个时候LT确实会带来一些性能损失。然而，实际上会存在很多多余的遍历么？

先不说第一次epoll_wait返回的时候，用户进程能否都将有数据返回的socket处理掉。在用户处理的过程中，如果该socket有新的数据上来，那么协议栈发现sk已经在ready_list中了，那么就不需要再次放入ready_list，也就是在LT模式下，对该sk的再次遍历不是多余的，是有效的。同时，我们回归epoll高效的场景在于，服务器有海量socket，但是活跃socket较少的情况下才会体现出epoll的高效、高性能。因此，在实际的应用场合，绝大多数情况下，ET模式在性能上并不会比LT模式具有压倒性的优势，至少，目前还没有实际应用场合的测试表面ET比LT性能更好。

**5.3.3 ET vs LT - 复杂度**

我们知道，对于可读事件而言，在阻塞模式下，是无法识别队列空的事件的，并且，事件通知机制，仅仅是通知有数据，并不会通知有多少数据。于是，在阻塞模式下，在epoll_wait返回的时候，我们对某个socket_fd调用recv或read读取并返回了一些数据的时候，我们不能再次直接调用recv或read，因为，如果socket_fd已经无数据可读的时候，进程就会阻塞在该socket_fd的recv或read调用上，这样就影响了IO多路复用的逻辑(我们希望是阻塞在所有被监控socket的epoll_wait调用上，而不是单独某个socket_fd上)，造成其他socket饿死，即使有数据来了，也无法处理。

接下来，我们只能再次调用epoll_wait来探测一些socket_fd，看是否还有数据可读。在LT模式下，如果socket_fd还有数据可读，那么epoll_wait就一定能够返回，接着，我们就可以对该socket_fd调用recv或read读取数据。然而，在ET模式下，尽管socket_fd还是数据可读，但是如果没有新的数据上来，那么epoll_wait是不会通知可读事件的。这个时候，epoll_wait阻塞住了，这下子坑爹了，明明有数据你不处理，非要等新的数据来了在处理，那么我们就死扛咯，看谁先忍不住。

等等，在阻塞模式下，不是不能用ET的么？是的，正是因为有这样的缺点，ET强制需要在非阻塞模式下使用。在ET模式下，epoll_wait返回socket_fd有数据可读，我们必须要读完所有数据才能离开。因为，如果不读完，epoll不会在通知你了，虽然有新的数据到来的时候，会再次通知，但是我们并不知道新数据会不会来，以及什么时候会来。由于在阻塞模式下，我们是无法通过recv/read来探测空数据事件，于是，我们必须采用非阻塞模式，一直read直到EAGAIN。因此，ET要求socket_fd非阻塞也就不难理解了。

另外，epoll_wait原本的语意是：监控并探测socket是否有数据可读(对于读事件而言)。LT模式保留了其原本的语意，只要socket还有数据可读，它就能不断反馈，于是，我们想什么时候读取处理都可以，我们永远有再次poll的机会去探测是否有数据可以处理，这样带来了编程上的很大方便，不容易死锁造成某些socket饿死。相反，ET模式修改了epoll_wait原本的语意，变成了：监控并探测socket是否有新的数据可读。

于是，在epoll_wait返回socket_fd可读的时候，我们需要小心处理，要不然会造成死锁和socket饿死现象。典型如listen_fd返回可读的时候，我们需要不断的accept直到EAGAIN。假设同时有三个请求到达，epoll_wait返回listen_fd可读，这个时候，如果仅仅accept一次拿走一个请求去处理，那么就会留下两个请求，如果这个时候一直没有新的请求到达，那么再次调用epoll_wait是不会通知listen_fd可读的，于是epoll_wait只能睡眠到超时才返回，遗留下来的两个请求一直得不到处理，处于饿死状态。

**5.3.4 ET vs LT - 总结**

最后总结一下，ET和LT模式下epoll_wait返回的条件

- ET - 对于读操作

[1] 当接收缓冲buffer内待读数据增加的时候时候(由空变为不空的时候、或者有新的数据进入缓冲buffer)

[2] 调用epoll_ctl(EPOLL_CTL_MOD)来改变socket_fd的监控事件，也就是重新mod socket_fd的EPOLLIN事件，并且接收缓冲buffer内还有数据没读取。(这里不能是EPOLL_CTL_ADD的原因是，epoll不允许重复ADD的，除非先DEL了，再ADD) 因为epoll_ctl(ADD或MOD)会调用sk的poll逻辑来检查是否有关心的事件，如果有，就会将该sk加入到epoll的ready_list中，下次调用epoll_wait的时候，就会遍历到该sk，然后会重新收集到关心的事件返回。

- ET - 对于写操作

[1] 发送缓冲buffer内待发送的数据减少的时候(由满状态变为不满状态的时候、或者有部分数据被发出去的时候) [2] 调用epoll_ctl(EPOLL_CTL_MOD)来改变socket_fd的监控事件，也就是重新mod socket_fd的EPOLLOUT事件，并且发送缓冲buffer还没满的时候。

- LT - 对于读操作 LT就简单多了，唯一的条件就是，接收缓冲buffer内有可读数据的时候
- LT - 对于写操作 LT就简单多了，唯一的条件就是，发送缓冲buffer还没满的时候

在绝大多少情况下，ET模式并不会比LT模式更为高效，同时，ET模式带来了不好理解的语意，这样容易造成编程上面的复杂逻辑和坑点。因此，建议还是采用LT模式来编程更为舒爽。

# 最后，Netty是个啥

## 从HTTP说起

传统的HTTP服务器的原理 

1. 创建一个ServerSocket，监听并绑定一个端口
2. 一系列客户端来请求这个端口
3. 服务器使用Accept，获得一个来自客户端的Socket连接对象 
4. 启动一个新线程处理连接 
	4.1. 读Socket，得到字节流
	4.2. 解码协议，得到Http请求对象 
	4.3. 处理Http请求，得到一个结果，封装成一个HttpResponse对象 
	4.4. 编码协议，将结果序列化字节流 写Socket，将字节流发给客户端 
5. 继续循环步骤3

HTTP服务器之所以称为HTTP服务器，是因为编码解码协议是HTTP协议，如果协议是Redis协议，那它就成了Redis服务器，如果协议是WebSocket，那它就成了WebSocket服务器，等等。 使用Netty你就可以定制编解码协议，实现自己的特定协议的服务器。

上面是一个传统处理http的服务器，但是在高并发的环境下，线程数量会比较多，System load也会比较高，于是就有了NIO。

他并不是Java独有的概念，NIO代表的一个词汇叫着IO多路复用。它是由操作系统提供的系统调用，早期这个操作系统调用的名字是select，但是性能低下，后来渐渐演化成了Linux下的epoll和Mac里的kqueue。我们一般就说是epoll，因为没有人拿苹果电脑作为服务器使用对外提供服务。而Netty就是基于Java NIO技术封装的一套框架。为什么要封装，因为原生的Java NIO使用起来没那么方便，而且还有臭名昭著的bug，Netty把它封装之后，提供了一个易于操作的使用模式和接口，用户使用起来也就便捷多了。

## Reactor线程模型

### **Reactor单线程模型**

一个NIO线程+一个accept线程：

![Reactor单线程模型](/blogImg/Reactor单线程模型.png)

### **Reactor多线程模型**

![Reactor多线程模型](/blogImg/Reactor多线程模型.png)

### **Reactor主从模型**

主从Reactor多线程：多个acceptor的NIO线程池用于接受客户端的连接

![Reactor主从模型](/blogImg/Reactor主从模型.png)

Netty可以基于如上三种模型进行灵活的配置。

## 总结

Netty是建立在NIO基础之上，Netty在NIO之上又提供了更高层次的抽象。

在Netty里面，Accept连接可以使用单独的线程池去处理，读写操作又是另外的线程池来处理。

Accept连接和读写操作也可以使用同一个线程池来进行处理。而请求处理逻辑既可以使用单独的线程池进行处理，也可以跟放在读写线程一块处理。线程池中的每一个线程都是NIO线程。用户可以根据实际情况进行组装，构造出满足系统需求的高性能并发模型

## netty的优点

如果不用netty，使用原生JDK的话，有如下问题：

1. API复杂
2. 对多线程很熟悉：因为NIO涉及到Reactor模式
3. 高可用的话：需要出路断连重连、半包读写、失败缓存等问题
4. JDK NIO的bug

而Netty来说，他的api简单、性能高而且社区活跃（dubbo、rocketmq等都使用了它）

![netty服务端创建时序图](/blogImg/netty服务端创建时序图.png)

![netty客户端时序图](/blogImg/netty客户端时序图.png)

# Netty解决的一些问题

## TCP 粘包/拆包

### 现象

先看如下代码，这个代码是使用netty在client端重复写100次数据给server端，ByteBuf是netty的一个字节容器，里面存放是的需要发送的数据

```java
public class FirstClientHandler extends ChannelInboundHandlerAdapter {    
    @Override    
    public void channelActive(ChannelHandlerContext ctx) {       
        for (int i = 0; i < 1000; i++) {            
            ByteBuf buffer = getByteBuf(ctx);            
            ctx.channel().writeAndFlush(buffer);        
        }    
    }    
    private ByteBuf getByteBuf(ChannelHandlerContext ctx) {        
        byte[] bytes = "你好，我的名字是1234567!".getBytes(Charset.forName("utf-8"));        
        ByteBuf buffer = ctx.alloc().buffer();        
        buffer.writeBytes(bytes);        
        return buffer;    
    }
}
```

从client端读取到的数据为：

![输出结果](/blogImg/输出结果.png)

从服务端的控制台输出可以看出，存在三种类型的输出

1. 一种是正常的字符串输出。
2. 一种是多个字符串“粘”在了一起，我们定义这种 ByteBuf 为粘包。
3. 一种是一个字符串被“拆”开，形成一个破碎的包，我们定义这种 ByteBuf 为半包。

### 透过现象分析原因

应用层面使用了Netty，但是对于操作系统来说，只认TCP协议，尽管我们的应用层是按照 ByteBuf 为 单位来发送数据，server按照Bytebuf读取，但是到了底层操作系统仍然是按照字节流发送数据，因此，数据到了服务端，也是按照字节流的方式读入，然后到了 Netty 应用层面，重新拼装成 ByteBuf，而这里的 ByteBuf 与客户端按顺序发送的 ByteBuf 可能是不对等的。因此，我们需要在客户端根据自定义协议来组装我们应用层的数据包，然后在服务端根据我们的应用层的协议来组装数据包，这个过程通常在服务端称为拆包，而在客户端称为粘包。

拆包和粘包是相对的，一端粘了包，另外一端就需要将粘过的包拆开，发送端将三个数据包粘成两个 TCP 数据包发送到接收端，接收端就需要根据应用协议将两个数据包重新组装成三个数据包。

### 如何解决

在没有 Netty 的情况下，用户如果自己需要拆包，基本原理就是不断从 TCP 缓冲区中读取数据，每次读取完都需要判断是否是一个完整的数据包 如果当前读取的数据不足以拼接成一个完整的业务数据包，那就保留该数据，继续从 TCP 缓冲区中读取，直到得到一个完整的数据包。 如果当前读到的数据加上已经读取的数据足够拼接成一个数据包，那就将已经读取的数据拼接上本次读取的数据，构成一个完整的业务数据包传递到业务逻辑，多余的数据仍然保留，以便和下次读到的数据尝试拼接。 

而在Netty中，已经造好了许多类型的拆包器，我们直接用就好：

![拆包器](/blogImg/拆包器.png)

选好拆包器后，在代码中client段和server端将拆包器加入到chanelPipeline之中就好了:

如上实例中：

客户端：

```java
ch.pipeline().addLast(new FixedLengthFrameDecoder(31));
```

服务端：

```java
ch.pipeline().addLast(new FixedLengthFrameDecoder(31));
```


![正常输出](/blogImg/正常输出.png)

## Netty 的零拷贝

### 传统意义的拷贝

是在发送数据的时候，传统的实现方式是：

1. `File.read(bytes)`

2. `Socket.send(bytes)`


这种方式需要四次数据拷贝和四次上下文切换：
1. 数据从磁盘读取到内核的read buffer
2. 数据从内核缓冲区拷贝到用户缓冲区
3. 数据从用户缓冲区拷贝到内核的socket buffer
4. 数据从内核的socket buffer拷贝到网卡接口（硬件）的缓冲区

### 零拷贝的概念

明显上面的第二步和第三步是没有必要的，通过java的FileChannel.transferTo方法，可以避免上面两次多余的拷贝（当然这需要底层操作系统支持）

1. 调用transferTo,数据从文件由DMA引擎拷贝到内核read buffer
2. 接着DMA从内核read buffer将数据拷贝到网卡接口buffer

上面的两次操作都不需要CPU参与，所以就达到了零拷贝。

### Netty中的零拷贝

主要体现在三个方面：

**1、bytebuffer**

Netty发送和接收消息主要使用bytebuffer，bytebuffer使用对外内存（DirectMemory）直接进行Socket读写。

原因：如果使用传统的堆内存进行Socket读写，JVM会将堆内存buffer拷贝一份到直接内存中然后再写入socket，多了一次缓冲区的内存拷贝。DirectMemory中可以直接通过DMA发送到网卡接口

**2、Composite Buffers**

传统的ByteBuffer，如果需要将两个ByteBuffer中的数据组合到一起，我们需要首先创建一个size=size1+size2大小的新的数组，然后将两个数组中的数据拷贝到新的数组中。但是使用Netty提供的组合ByteBuf，就可以避免这样的操作，因为CompositeByteBuf并没有真正将多个Buffer组合起来，而是保存了它们的引用，从而避免了数据的拷贝，实现了零拷贝。

**3、对于FileChannel.transferTo的使用**

Netty中使用了FileChannel的transferTo方法，该方法依赖于操作系统实现零拷贝。

