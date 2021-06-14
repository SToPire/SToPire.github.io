---
title: XPC Architectural Support for Secure and Efficient Cross Process Call
date: 2021-06-15 00:59:06
categories: paper
tags: [operating system, microkernel, inter-process communication, accelerators]
---
### 名词解释

#### capability

文章中多次出现的capability是linux的细粒度权限控制手段，是每个进程（线程）的属性。特权操作被细分成组，进程（线程）执行对应操作需要拥有对应的capability.

<!-- more -->

#### domain

protected domain是对象和能对其进行的操作(object, {set of operations on object})构成的集合，domain switching就是改变有权操作的对象。比如地址空间切换就是一种domain switch，也可以依赖tagged memory之类的手段在一个地址空间内实现隔离和domain switch：此时domain不再是地址空间隔离出的进程，而是有自己ID的，统一地址空间中的”对象及操作“集合。

#### trampoline

又称indirect jump vectors，喻指跳往函数调用/中断例程等入口点的代码，类似于一个wrapper。进入trampoline执行逻辑后再跳出到原来的入口点，所以得名蹦床。

#### TOCTTOU

朴素的message passing需要两次拷贝，即caller->kernel->callee. 如果通过caller->shared buffer，再将shared buffer共享给callee的方式实现的话，恶意caller可能在callee check完消息之后修改它，即Time Of Check To Time Of Use.

### 动机

进程间通信(IPC)在多数内核中都是一项昂贵的动作，尤其是对微内核的影响尤甚，因为它的很多模块放在用户空间中，依赖IPC通信。IPC中大部分时间都在做两件事：domain switching和message copying：

1. 现有的大部分实现都要求domain switching陷入内核态来实现地址空间的切换。
2. message transfer虽然可以使用共享内存，但是做不到零拷贝，因为在原址上的update不够安全，caller可以观察到callee的动作甚至通过修改共享内存来影响callee. 在现有的实现中，即使采用共享内存，也需要将message重新拷贝到共享内存中。

基于以上两点，本文提出了XPC(Cross Process Call)，这是一种不依赖陷入内核直接进行domain checking，进行零拷贝的message passing，同时最小化对硬件的修改，从而可以轻松集成到现有内核的操作系统原语。



### 设计

#### domain switching

1. 一条x-entry关联到一个具体的procedure，在server进程完成注册后，x-entry就被存入**全局**的x-entry表中，进程可以通过其id指明调用的x-entry，从而指明调用哪个过程。
2. xcall-cap是进程本地的bitmap，指明线程对于哪些x-entry有调用权限。
3. linkage record是对调用返回所需信息（用户态拿不到的信息）的统称，在xcall时存放在进程本地的link stack中，xret时根据其信息返回到caller.
4. 调用xcall时，会先检查xcall-cap，再取出x-entry并检查是否valid，然后保存linkage record供返回使用，最后切换页表寄存器以切换地址空间。**这相当于在client的上下文里运行server的代码。**
5. 鉴于IPC调用的时间局部性和可预测性，可以为x-entry准备一个cache并预取。

#### message passing

1. seg-reg寄存器存着一段连续虚拟内存和连续物理内存之间的映射，称为中继段(relay-seg)。这个寄存器内容可以直接在xcall时传递，实现零拷贝的message passing.
2. 用户态进程不能随意修改seg-reg，所以它们可以用seg-mask遮盖这一段内存的一部分，如果IPC调用下游进程只需要传递消息的一部分，就可以采用这种方式。
3. 进程有一个本地的seg-list，可以存多个relay-seg，当进程希望使用另一个relay-seg来发起IPC调用时，可以使用`swapseg #reg`指令，这是一个交换seg-reg中与seg-list某一位置的relay-seg的原子操作。
4. 为了防止TOCTTOU出现，内核确保一条relay-seg在同一时刻只在一个CPU核心上活跃。
5. callee从xret返回时需要检查seg-reg是否发生了变化，防止恶意callee从自己的seg-list里换了一个不同的relay-seg给caller.



### 实现

#### 微内核

1. 引入了名为grant_cap的capability，当进程创建x-entry时，它初始拥有该x-entry的grant权限，这一权限允许它将与该x-entry关联的xcall或者grant权限授予其他进程。
2. 在用户空间进行的domain switch对内核而言是透明的，比如线程A使用xcall调用B，线程B触发page fault陷入内核。内核不知道当前运行线程是B，于是使用A的页表处理page fault. 为了解决这个问题，我们把线程的状态分裂成scheduling state和runtime state，前者包含线程优先级等调度需要的信息，后者包含地址空间等内核为线程提供服务所需的信息。每个线程关联到唯一的scheduling state，但可以有多个runtime state. 内核根据runtime state（在XPC中，由xcall-cap-reg决定）为线程提供服务。
3. server的一条x-entry可以同时被多个client调用。为了实现这一点，创建x-entry时需要指定同时调用者的最大数目，创建好对应数目的XPC context。每个x-entry有一个trampoline，寻找一个空闲的XPC context，切换到它的execution stack上并准备好本地data，然后开始执行x-entry对应procedure的逻辑。invocation结束并返回前，trampoline负责释放XPC context资源。这样的设计容易遭到DoS攻击，所以需要防止恶意client多次调用x-entry，把所有context占满的行为。
4. 调用链中某个进程可能意外终止，此时需要优雅的方式抛出exception。假设调用链A->B->C，在XPC中，如果B意外终止，内核会扫描所有的link stack并把与B相关的条目标为invalid。这样一来，从C返回到B时硬件会抛出异常，内核可以做出处理（如pop出B的linkage record，并返回一个timeout error给A)。另一种做法是把B的页表基地址清零，C返回到B时会产生page fault并由内核处理，避免了扫描所有link stack的过程。

#### Android Binder

1. 保持Android Binder Framework提供的`transact()`和`onTransact()`接口不变，在Linux Binder driver中添加XPC的相关接口，修改Android Binder Framework使其依靠XPC的接口进行进程间调用。依赖relay-segment来做零拷贝的消息传递。
2. ashmem(anonymous shared momory)是Android提供的，用文件指向匿名内存并通过传递文件描述符实现共享内存的接口。这个实现需要一次额外拷贝(shared buffer->callee)来避免TOCTTOU问题。可以依靠XPC的relay-seg实现ashmem免去这一次拷贝，但是对于应用同时需要使用多个ashmem的情形需要进行relay-seg之间的切换。

#### IPC调用链

以A->B->C调用为例，实现零拷贝的handover有一些问题需要解决：

1. B可能需要在A传来的relay-seg后面添加内容再传给C.（如网络栈中上层协议给payload加header），这种情况下需要调用链中各进程沟通message size.
2. B可能需要把A传来的relay-seg的一部分传给C.（如文件系统把请求分成block传给设备驱动），这种情况下依赖seg-mask寄存器遮盖relay-seg再进行xcall.
3. 回收一个relay-seg：进程终止时，内核扫描其seg-list并把其caller的relay-seg返回去，回收其他的relay-seg.

### 疑问

1. xcall时，linkage record是被保存在(caller/callee)的link stack中？按照3.2节对xcall过程的介绍似乎应该是在caller的栈里，但是link-reg寄存器是怎么在domain switch时保存，恢复时callee又是怎么拿到caller地址空间中的linkage record呢？

   猜想：link stack作为每线程的数据结构，内核在上下文切换时负责保存和恢复它，即保存和恢复link-reg寄存器。既然link stack是由kernel管理的，可能kernel知道各个线程link stack的位置，正如上下文切换时kernel能从各个进程PCB(线程TCB)中保存和读取上下文一样。
