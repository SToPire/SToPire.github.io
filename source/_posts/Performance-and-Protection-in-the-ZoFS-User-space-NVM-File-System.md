---
title: Performance and Protection in the ZoFS User-space NVM File System
date: 2021-06-13 12:15:42
categories: paper
tags: [non-volatile memory, user-space file systems, memory protection keys]
---

### 名词解释

#### NVM

Non-volatile memory（非易失性存储），类似硬盘，存储的信息不会随着断电而消失；但又接在内存总线之上，也就是可以按字节寻址，并被load/store指令直接读写。Inter的傲腾系列就是NVM的一项商业应用。既然是非易失性存储，就有在其上构建文件系统的意义和动机。

<!-- more -->

#### User-space file system

一般意义上的文件系统是操作系统的一部分，自然工作在内核态。用户态文件系统，顾名思义，指文件系统的data和metadata都由用户态进程提供。这样做方便用户创建和调试自定义的文件系统而不用重新编译内核，缺点就是多次用户态/内核态的切换严重影响性能。

用户态文件系统并不意味着其工作不需要内核的参与：Linux中对文件的访问都要走VFS的接口，VFS判断出请求的是一个用户态文件系统后，就会转交给名为FUSE(filesystem in userspace)的**内核**模块，它是内核提供给用户态文件系统并与之通信的服务接口，参见[这里](https://zhuanlan.zhihu.com/p/106719192)。

#### User-space NVMFS

现有的现允许直接在用户态读数据，但是写数据尤其是写metadata是受限的，比如要IPC请求一个特权进程来做，或者写下log来消费。

#### Memory Protection Key

一种管理内存访问权限的方式。每个页表entry被打上region number，另有PKRU寄存器存储每个region number的访问权限。为修改访问权限，进程可以使用指令修改PKRU寄存器，而传统的page permission需要大量的系统调用并会导致TLB失效。这个机制并不是为安全而设计的，因此修改PKRU并不是特权指令。

### 面临的问题

在当前的实现中，NVM之上的user-space FS library不能同时操控data和metadata，这是因为：

1. 文件系统和应用程序都在用户态地址空间中，应用程序可以绕过文件系统的permission check
2. 应用程序的bug可能破坏文件系统的metadata，导致整个文件系统的瘫痪。

### 本文的工作

1. 提出了*coffer*，它是一些NVM page的集合，存储了拥有相同permission的文件。这是基于一项观察：应用往往将相同permission的文件放在一起，且很少修改permission.
2. 提出了基于coffer的文件系统架构*Treasury*，将protection和management分开：用户态文件系统library可以在coffer内进行任何修改，而跨coffer的操作受到Treasury内核(KernFS)的保护。KernFS工作在内核态，维护各coffer的metadata并对coffer一级的访问做鉴权。Treasury同时提供了一般文件系统的特性，如文件描述符映射表和符号连接，从而使得应用不用重新编译就能在其上运行。
3. 提出了基于Treasury的文件系统*ZoFS*.

#### Treasury如何做到隔离和保护

1. Stray write，即对错误内存位置的写操作。常用的做法有依赖页表permission位，写窗口等等。Treasury拓展了写窗口的思想，依靠MPK把coffer的页从进程地址空间中标注出来，写coffer之前必须修改PKRU寄存器获得权限，写完再把权限还回去。因为权限是在寄存器（而不是页表permission位）管理的，这个方法也防止了同一进程下多个线程之间的stray write.
2. Graceful error return. 内核态的文件系统在系统调用出错时可以返回错误码，但用户态文件系统如果访问了不合法地址（如一个崩溃的coffer）则会触发SIGSEGV导致整个进程被杀死。本文的做法是hook掉segmentation fault handler，把fault翻译回文件系统的业务错误码，用一个setlongjump跳回到文件系统函数里返回错误。
3. Fault isolation. 同一时刻，用户空间中至多只有一个coffer是可访问的，这就要求KernFS给映射到同一个进程的coffer分配不同的MPK编号（至多15个）。进程要想访问另一个coffer，只要修改PKRU寄存器。
4. 保护文件系统metadata. 恶意进程可以修改一个共享coffer的metadata，实现对其他进程的攻击。但是对一个coffer的破坏不会扩散到其他coffer（如企图访问其他coffer的数据），这是因为fault isolation的存在，进程访问不了其他的coffer，除非恶意进程操纵了一条合法的跨coffer引用。为了防御这种攻击，进程遇到coffer引用时，要在改变coffer accessibility之前检查引用的合法性：比如目标coffer的文件系统路径是否符合预期，是否指向一个coffer的root inode等等。

### 实现细节

1. 维护了一个从用户态file descriptor到内核fd的映射，通过查表确定一个fd指向了FSLib中的文件还是内核文件系统中的文件。
2. 通过lease lock配合线程本地free pages list的方式实现高争用的页分配器。线程尝试用lease lock获取free page list，在其未过期的时候都可以本地分配，否则从共享池获取一个list到线程本地。但是在分配特别频繁的情况下，一个free page list不够，线程会频繁调用`coffer_enlarge`向kernel获取更多page，这个操作本身可能会在内核产生争用（见6.1节）。
3. ZoFS的目录结构是一个二级哈希表，第一级是指向第二级page的指针，第二级page有一半直接放dentry（针对小目录），另一半是二级哈希表，每个桶里放dentry。这个目录结构对于包含很多文件的大目录也有良好性能。但是对于目录小而路径深的case，ZoFS的性能不佳，因为为了找到一个coffer root page，长路径的每一个前缀（从最长开始）都要被检查。