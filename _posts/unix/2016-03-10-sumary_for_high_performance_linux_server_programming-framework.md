---
layout: post
title: "高性能服务器程序框架"
category: "unp"
tags: [unp,Unix网络编程,框架]
---
>《Linux高性能服务器编程》读书笔记之。高性能服务器程序框架是服务器编程的核心部分。

将服务器解构为如下三个主要模块：

- I/O处理单元。包含：四种I/O模型和两种高效事件处理模式
- 逻辑单元。包含：两种高效并发模式，以及高效的逻辑处理方式—有限状态机
- 存储单元。如db，文件等

将按照如下流程去学习：

- 服务器模型
- 服务器编程框架
- I/O模型
- 并发模式

如下思维导图：


![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-summary.png)

# 一.服务器模型
分为：

- C/S模型（大多数）
- P2P模型（少数，点对点）

## 1. C/S模型
图1 CS模型

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-1.jpg)

C/S模型的逻辑很简单。服务器启动后，首先创建一个或多个监听socket，并调用`bind`函数将其绑定到服务器指定的端口上，然后调用`listen`函数等待客户连接。服务器稳定运行之后，客户端就可以调用`connect`函数向服务器发起连接了。

由于客户连接请求是随机到达的异步事件，服务器需要使用某种**I/O模型**来监听这一**事件**。I/O模型有多种：如下是其中的一种。

*图2 I/O复用技术之一的select系统调用*

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-2.jpg)

上图分析如下：

- 当监听到连接请求后，服务器就调用`accept`函数接受它，并分配一个**逻辑单元**为新的连接服务。
- **逻辑单元可以是新创建的子进程、子线程或者其他**。上图服务器给客户端分配的逻辑单元是由fork系统调用创建的**子进程**。
- 逻辑单元读取客户请求，处理该请求，然后将处理结果返回给客户端。
- 客户端接收到服务器反馈的结果之后，可以继续向服务器发送请求，也可以立即主动关闭连接。
- 如果客户端主动关闭连接，则服务器执行被动关闭连接。至此，双方的通信结束。
- 需要注意的是，服务器在处理一个客户请求的同时还会继续监听其他客户请求，否则就变成了效率低下的串行服务器了（必须先处理完前一个客户的请求，才能继续处理下一个客户请求)

**CS模型的优缺点：**

>C/S模型非常**适合资源相对集中的场合**，并且它的实现也很简单，但其缺点也很明显：**服务器是通信的中心，当访问量过大时，可能所有客户都将得到很慢的响应**。下面讨论的P2P模型解决了这个问题。

## 2. P2P模型
P2P（Peer to Peer，点对点）模型比C/S模型更符合网络通信的实际情况。**它摒弃了以服务器为中心的格局，让网络上所有主机重新回归对等的地位。**

*图3 P2P模型*

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-3.jpg)

**P2P模型的优缺点：**

- 优点：使得每台机器在消耗服务的同时也给别人提供服务，这样资源能够充分、自由地共享。云计算机群可以看作P2P模型的一个典范。
- 缺点：当用户之间传输的请求过多时，网络的负载将加重。

从编程角度来讲，P2P模型可以看作C/S模型的扩展：每台主机既是客户端，又是服务器。因此，我们仍然采用C/S模型来讨论网络编程。

# 二. 服务器编程框架
*图4 服务器基础框架*


![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-4.jpg)

该图既能用来描述一台服务器，也能用来描述一个服务器机群。两种情况下各个部件的含义和功能如表所示:

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-6.jpg)

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-7.jpg)

## 1. I/O处理单元
I/O处理单元是**服务器管理客户连接的模块**。

它通常要完成以下工作：

- 等待并接受新的客户连接
- 接收客户数据
- 将服务器响应数据返回给客户端。

但是，**数据的收发不一定在I/O处理单元中执行，也可能在逻辑单元中执行，具体在何处执行取决于事件处理模式。**

对于一个服务器机群来说，I/O处理单元是一个专门的接入服务器。它实现负载均衡，从所有逻辑服务器中选取负荷最小的一台来为新客户服务。（相当于负载均衡服务器）

## 2.逻辑单元

**一个逻辑单元通常是一个进程或线程。**

它的工作：

- 分析并处理客户数据
- 将结果传递给I/O处理单元或者直接发送给客户端（具体使用哪种方式取决于事件处理模式）
- 对服务器机群而言，一个逻辑单元本身就是一台逻辑服务器。服务器通常拥有多个逻辑单元，以实现对多个客户任务的并行处理。

## 3.网络存储单元

**网络存储单元可以是数据库、缓存和文件，甚至是一台独立的服务器。但它不是必须的**，比如ssh、telnet等登录服务就不需要这个单元。

## 4.请求队列
**请求队列是各单元之间的通信方式的抽象。**

I/O处理单元接收到客户请求时，需要以某种方式通知一个逻辑单元来处理该请求。同样，多个逻辑单元同时访问一个存储单元时，也需要采用某种机制来协调处理竞态条件。请求队列通常被实现为**池**的一部分。

对于服务器机群而言，请求队列是各台服务器之间预先建立的、静态的、永久的TCP连接。这种TCP连接能提高服务器之间交换数据的效率，因为它避免了动态建立TCP连接导致的额外的系统开销。

# 三. I/O模型
>是服务器编程框架中I/O处理单元的一个细分

从两大方面开始：

- 阻塞与非阻塞
- 同步与异步

在知乎回答:["怎样理解阻塞非阻塞与同步异步的区别？"](https://www.zhihu.com/question/19732473)中理解两者的概念和区别。

## 1. 阻塞与非阻塞
**基础socket默认是阻塞的**，我们可以给socket系统调用的第2个参数传递`SOCK_NONBLOCK`标志，或者通过fcntl系统调用的`F_SETFL`命令，将其设置为**非阻塞**的。

>注：**阻塞和非阻塞的概念能应用于所有文件描述符，而不仅仅是socket。我们称阻塞的文件描述符为阻塞I/O，称非阻塞的文件描述符为非阻塞I/O。**

### 1.1 阻塞I/O
针对阻塞I/O执行的系统调用可能因为无法立即完成而被操作系统挂起，直到等待的事件发生为止
socket的基础API中，可能被阻塞的系统调用包括`accept`、`send`、`recv`和`connect`。

### 1.2 非阻塞I/O
**针对非阻塞I/O执行的系统调用则总是立即返回，而不管事件是否已经发生。**

如果事件没有立即发生，这些系统调用就返回-1，和出错的情况一样。此时我们必须根据errno来区分这两种情况。对`accept`、`send`和`recv`而言，事件未发生时errno通常被设置成`EAGAIN`（意为“再来一次”）或者`EWOULDBLOCK`（意为“期望阻塞”）；对`connect`而言，errno则被设置成`EINPROGRESS`（意为“在处理中”）。

**显然，我们只有在事件已经发生的情况下操作非阻塞I/O（读、写等），才能提高程序的效率。因此，非阻塞I/O通常要和其他I/O通知机制一起使用，比如I/O复用和SIGIO信号。**

#### 1). I/O复用
I/O复用是最常使用的I/O通知机制。

>**I/O复用使得程序能同时监听多个文件描述符。应用程序通过I/O复用函数向内核注册一组事件，内核通过I/O复用函数把其中就绪的事件通知给应用程序。**

网络编程中使用I/O复用的场景：

- 客户端程序需要同时处理多个socket时，如非阻塞connect程序
- 客户端程序同时处理用户输入和网络连接时，如聊天室程序
- TCP服务器要同时处理监听socket和连接socket时 (这是I/O复用使用最多的场合)
- 服务器要同时处理TCP请求和UDP请求时


Linux上常用的I/O复用函数是**`select`**、**`poll`**和**`epoll_wait`**

**需要指出的是，I/O复用函数本身的调用是阻塞的，它们能提高程序效率的原因在于它们具有同时监听多个I/O事件的能力。**

#### 2).SIGIO信号（信号驱动I/O）
SIGIO信号也可以用来报告I/O事件

我们可以为一个目标文件描述符指定**宿主进程**，那么被指定的宿主进程将捕获到`SIGIO`信号。这样，当目标文件描述符上有事件发生时，`SIGIO`信号的信号处理函数将被触发，我们也就可以在该信号处理函数中对目标文件描述符执行非阻塞I/O操作了。

## 2. 同步与异步
**从理论上说，阻塞I/O、I/O复用和信号驱动I/O都是同步I/O模型。因为在这三种I/O模型中，I/O的读写操作，都是在I/O事件发生之后，由应用程序来完成的。**

而POSIX规范所定义的异步I/O模型则不同:

>对异步I/O而言，用户可以直接对I/O执行读写操作，这些操作告诉内核 『用户读写缓冲区的位置，以及I/O操作完成之后内核通知应用程序的方式。』

**异步I/O的读写操作总是立即返回，而不论I/O是否是阻塞的，因为真正的读写操作已经由内核接管。**

也就是说：

- 同步I/O模型要求用户代码自行执行I/O操作（将数据从内核缓冲区读入用户缓冲区，或将数据从用户缓冲区写入内核缓冲区）
- 异步I/O机制则由内核来执行I/O操作（数据在内核缓冲区和用户缓冲区之间的移动是由内核在“后台”完成的）。

你也可以这样认为：

- 同步I/O向应用程序通知的是**I/O就绪事件**
- 异步I/O向应用程序通知的是**I/O完成事件**

Linux环境下，`aio.h`头文件中定义的函数提供了对异步I/O的支持

4种I/O模型的差异：


![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-8.jpg)

# 四.两种高效的事件处理模式
>也是服务器编程框架中I/O处理单元的一个细分

服务器程序通常需要处理三类事件：

- I/O事件
- 信号
- 定时事件

有两种高效的事件处理模式：`Reactor`和`Proactor`。

同步I/O模型通常用于实现Reactor模式，异步I/O模型则用于实现Proactor模式。

## 1.Reactor模式
>Reactor是这样一种模式，它要求主线程（I/O处理单元，下同）只负责监听文件描述上是否有事件发生，有的话就立即将该事件通知工作线程（逻辑单元，下同）。除此之外，主线程不做任何其他实质性的工作。读写数据，接受新的连接，以及处理客户请求均在工作线程中完成。

使用同步I/O模型（以`epoll_wait`为例）实现的Reactor模式的工作流程是：

- 主线程往epoll内核事件表中注册socket上的读就绪事件
- 主线程调用`epoll_wait`等待socket上有数据可读
- 当socket上有数据可读时，`epoll_wait`通知主线程。主线程则将socket可读事件放入请求队列。
- 睡眠在请求队列上的某个工作线程被唤醒，它从socket读取数据，并处理客户请求，然后往epoll内核事件表中注册该socket上的写就绪事件
- 主线程调用`epoll_wait`等待socket可写。
- 当socket可写时，`epoll_wait`通知主线程。主线程将socket可写事件放入请求队列。
- 睡眠在请求队列上的某个工作线程被唤醒，它往socket上写入服务器处理客户请求的结果。

*图5：Reactor模式的工作流程*

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-9.jpg)

上图工作线程从请求队列中取出事件后，将根据事件的类型来决定如何处理它：对于可读事件，执行读数据和处理请求的操作；对于可写事件，执行写数据的操作。因此，图所示的Reactor模式中，没必要区分所谓的“读工作线程”和“写工作线程”。

## 2.Proactor模式
与Reactor模式不同，**Proactor模式将所有I/O操作都交给主线程和内核来处理，工作线程仅仅负责业务逻辑**。因此，Proactor模式更符合图4所描述的服务器编程框架。

使用异步I/O模型（以`aio_read`和`aio_write`为例）实现的Proactor模式的工作流程是：

- 主线程调用`aio_read`函数向内核注册socket上的读完成事件，并告诉内核用户读缓冲区的位置，以及读操作完成时如何通知应用程序
- 主线程继续处理其他逻辑。
- 当socket上的数据被读入用户缓冲区后，内核将向应用程序发送一个信号，以通知应用程序数据已经可用。
- 应用程序预先定义好的信号处理函数选择一个工作线程来处理客户请求。工作线程处理完客户请求之后，调用`aio_write`函数向内核注册socket上的写完成事件，并告诉内核用户写缓冲区的位置，以及写操作完成时如何通知应用程序（仍然以信号为例）。
- 主线程继续处理其他逻辑。
- 用户缓冲区的数据被写入socket之后，内核将向应用程序发送一个信号，以通知应用程序数据已经发送完毕。
- 应用程序预先定义好的信号处理函数选择一个工作线程来做善后处理，比如决定是否关闭socket。

*图6:Proactor模式的工作流程*


![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-10.jpg)

图6，连接socket上的读写事件是通过`aio_read`/`aio_write`向内核注册的，因此内核将通过信号来向应用程序报告连接socket上的读写事件。所以，主线程中的`epoll_wait`调用仅能用来检测监听socket上的连接请求事件，而不能用来检测连接socket上的读写事件。

## 3.使用同步I/O方式模拟Proactor模式
其原理是：主线程执行数据读写操作，读写完成之后，主线程向工作线程通知这一“完成事件”。那么从工作线程的角度来看，它们就直接获得了数据读写的结果，接下来要做的只是对读写的结果进行逻辑处理。
使用同步I/O模型（仍然以`epoll_wait`为例）模拟出的Proactor模式的工作流程如下：

- 主线程往epoll内核事件表中注册socket上的读就绪事件。
- 主线程调用`epoll_wait`等待socket上有数据可读。
- 当socket上有数据可读时，`epoll_wait`通知主线程。主线程从socket循环读取数据，直到没有更多数据可读，然后将读取到的数据封装成一个请求对象并插入请求队列。
- 睡眠在请求队列上的某个工作线程被唤醒，它获得请求对象并处理客户请求，然后往epoll内核事件表中注册socket上的写就绪事件。
- 主线程调用`epoll_wait`等待socket可写。
- 当socket可写时，`epoll_wait`通知主线程。主线程往socket上写入服务器处理客户请求的结果。

*图7 同步I/O模型模拟出的Proactor模式的工作流程*


![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-11.jpg)

# 五.两种高效的并发模式
>是逻辑单元的一个细分

**并发编程的目的是让程序“同时”执行多个任务。**

如果程序是计算密集型的，并发编程并没有优势，反而由于任务的切换使效率降低。但如果程序是I/O密集型的，比如经常读写文件，访问数据库等，则情况就不同了。

由于I/O操作的速度远没有CPU的计算速度快，所以让程序阻塞于I/O操作将浪费大量的CPU时间。如果程序有多个执行线程，则当前被I/O操作所阻塞的执行线程可主动放弃CPU（或由操作系统来调度），并将执行权转移到其他线程。这样一来，CPU就可以用来做更加有意义的事情（除非所有线程都同时被I/O操作所阻塞），而不是等待I/O操作完成，因此CPU的利用率显著提升。

- 从实现上来说，**并发编程主要有多进程和多线程两种方式**。
- 服务器主要有两种并发编程模式：**半同步/半异步（half-sync/half-async）模式和领导者/追随者（Leader/Followers）模式。**

## 1.半同步/半异步模式
首先，半同步/半异步模式中的“同步”和“异步”与前面讨论的I/O模型中的“同步”和“异步”是完全不同的概念。

- 在I/O模型中，“同步”和“异步”区分的是内核向应用程序通知的是何种I/O事件（是就绪事件还是完成事件），以及该由谁来完成I/O读写（是应用程序还是内核）
- 在并发模式中，“同步”指的是程序完全按照代码序列的顺序执行；“异步”指的是程序的执行需要由系统事件来驱动。常见的系统事件包括中断、信号等。

*图8.并发模式中的同步和异步读操作*

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-12.jpg)

那么什么是半同步/半异步模式呢？？

按照同步方式运行的线程称为同步线程，按照异步方式运行的线程称为异步线程。

- 显然，异步线程的执行效率高，实时性强，这是很多嵌入式程序采用的模型。但编写以异步方式执行的程序相对复杂，难于调试和扩展，而且不适合于大量的并发。
- 同步线程则相反，它虽然效率相对较低，实时性较差，但逻辑简单。

因此，对于像服务器这种既要求较好的实时性，又要求能同时处理多个客户请求的应用程序，我们就应该同时使用同步线程和异步线程来实现，即采用半同步/半异步模式来实现。

半同步/半异步模式对应网络编程模型：

- 同步线程用于处理客户逻辑，相当于图4中的逻辑单元；
- 异步线程用于处理I/O事件，相当于图4中的I/O处理单元。

流程：

- 异步线程监听到客户请求后，就将其封装成请求对象并插入请求队列中
- 请求队列将通知某个工作在同步模式的工作线程来读取并处理该请求对象。

具体选择哪个工作线程来为新的客户请求服务，则取决于请求队列的设计。比如最简单的轮流选取工作线程的Round Robin算法，也可以通过条件变量或信号量来随机地选择一个工作线程。图9总结了半同步/半异步模式的工作流程。

*图9:半同步/半异步模式的工作流程*

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-13.jpg)

### (1).半同步/半反应堆（half-sync/half-reactive）模式
**是半同步/半异步模式的变体**

工作方式：

- 异步线程只有一个，由主线程来充当， 负责监听所有socket上的事件
- 如果监听socket上有可读事件发生(即有新的连接请求到来)，主线程就接受之以得到新的连接socket，然后往epoll内核事件表中注册该socket上的读写事件。
- 如果连接socket上有读写事件发生(即有新的客户请求到来或有数据要发送至客户端)，主线程就将该连接socket插入请求队列中
- 所有工作线程都sleep在请求队列上，当有任务到来时，它们将通过竞争（比如申请互斥锁）获得任务的接管权。这种竞争机制使得只有空闲的工作线程才有机会来处理新任务，这是很合理的。

*图10：半同步/半反应堆模式*

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-14.jpg)

图10中，主线程插入请求队列中的任务是就绪的连接socket。这说明该图所示的半同步/半反应堆模式采用的事件处理模式是Reactor模式：它要求工作线程自己从socket上读取客户请求和往socket写入服务器应答。这就是该模式的名称中“half-reactive”的含义。

实际上，半同步/半反应堆模式也可以使用模拟的Proactor事件处理模式，即由主线程来完成数据的读写。在这种情况下，主线程一般会将应用程序数据、任务类型等信息封装为一个任务对象，然后将其（或者指向该任务对象的一个指针）插入请求队列。工作线程从请求队列中取得任务对象之后，即可直接处理之，而无须执行读写操作了。

**半同步/半反应堆模式存在如下缺点：**

- 主线程和工作线程共享请求队列。主线程往请求队列中添加任务，或者工作线程从请求队列中取出任务，都需要对请求队列加锁保护，从而白白耗费CPU时间。
- 每个工作线程在同一时间只能处理一个客户请求。如果客户数量较多，而工作线程较少，则请求队列中将堆积很多任务对象，客户端的响应速度将越来越慢。如果通过增加工作线程来解决这一问题，则工作线程的切换也将耗费大量CPU时间。

**一种相对高效的半同步/半异步模式**：它的每个工作线程都能同时处理多个客户连接

*图11 高效的半同步/半异步模式*

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-15.jpg)


主线程只管理监听socket，连接socket由工作线程来管理。当有新的连接到来时，主线程就接受之并将新返回的连接socket派发给某个工作线程，此后该新socket上的任何I/O操作都由被选中的工作线程来处理，直到客户关闭连接。主线程向工作线程派发socket的最简单的方式，是往它和工作线程之间的管道里写数据。工作线程检测到管道上有数据可读时，就分析是否是一个新的客户连接请求到来。如果是，则把该新socket上的读写事件注册到自己的epoll内核事件表中。

每个线程（主线程和工作线程）都维持自己的事件循环，它们各自独立地监听不同的事件。因此，在这种高效的半同步/半异步模式中，每个线程都工作在异步模式，所以它并非严格意义上的半同步/半异步式中，每个线程都工作在异步模式，所以它并非严格意义上的半同步/半异步模式。

## 2.领导者/追随者模式
领导者/追随者模式是多个工作线程轮流获得事件源集合，轮流监听、分发并处理事件的一种模式。

在任意时间点，程序都仅有一个领导者线程，它负责监听I/O事件。而其他线程则都是追随者，它们休眠在线程池中等待成为新的领导者。

当前的领导者如果检测到I/O事件，首先要从线程池中推选出新的领导者线程，然后处理I/O事件。此时，新的领导者等待新的I/O事件，而原来的领导者则处理I/O事件，二者实现了并发。

领导者/追随者模式包含如下几个组件：

- 句柄集（HandleSet）
- 线程集（ThreadSet）
- 事件处理器（EventHandler）
- 具体的事件处理器（ConcreteEventHandler）

它们的关系如图12所示


![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-16.jpg)

### (1).句柄集
句柄（Handle）用于表示I/O资源，在Linux下通常就是一个文件描述符。

句柄集管理众多句柄，它使用`wait_for_event`方法来监听这些句柄上的I/O事件，并将其中的就绪事件通知给领导者线程。领导者则调用绑定到Handle上的事件处理器来处理事件。领导者将Handle和事件处理器绑定是通过调用句柄集中的`register_handle`方法实现的。

### (2).线程集
这个组件是所有工作线程（包括领导者线程和追随者线程）的管理者。它负责各线程之间的同步，以及新领导者线程的推选。

线程集中的线程在任一时间必处于如下三种状态之一：

- Leader：线程当前处于领导者身份，负责等待句柄集上的I/O事件。
- Processing：线程正在处理事件。领导者检测到I/O事件之后，可以转移到Processing状态来处理该事件，并调用`promote_new_leader`方法推选新的领导者；也可以指定其他追随者来处理事件（Event Handoff），此时领导者的地位不变。当处于Processing状态的线程处理完事件之后，如果当前线程集中没有领导者，则它将成为新的领导者，否则它就直接转变为追随者。
- Follower：线程当前处于追随者身份，通过调用线程集的join方法等待成为新的领导者，也可能被当前的领导者指定来处理新的任务。

*图13显示了这三种状态之间的转换关系*

![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-17.jpg)

>需要注意的是，领导者线程推选新的领导者和追随者等待成为新领导者这两个操作都将修改线程集，因此线程集提供一个成员Synchronizer来同步这两个操作，以避免竞态条件。

### (3).事件处理器和具体的事件处理器
事件处理器通常包含一个或多个回调函数`handle_event`。

这些回调函数用于处理事件对应的业务逻辑。事件处理器在使用前需要被绑定到某个句柄上，当该句柄上有事件发生时，领导者就执行与之绑定的事件处理器中的回调函数。具体的事件处理器是事件处理器的派生类。它们必须重新实现基类的`handle_event`方法，以处理特定的任务。

*图14领导者/追随者模式的工作流程*


![](https://raw.githubusercontent.com/BeginMan/beginman.github.io/master/img/in-post/unp-18.jpg)

由于领导者线程自己监听I/O事件并处理客户请求，因而领导者/追随者模式不需要在线程之间传递任何额外的数据，也无须像半同步/半反应堆模式那样在线程之间同步对请求队列的访问。但领导者/追随者的一个明显缺点是仅支持一个事件源集合，因此也无法像图11所示的那样，让每个工作线程独立地管理多个客户连接。
