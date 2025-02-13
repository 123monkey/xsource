#临界知识
[](https://zhuanlan.zhihu.com/p/83398714)
[](https://time.geekbang.org/column/article/232676)
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/f07a5823.png)
```asp
磁盘寻址速度:ms,磁盘读写速度:1G/s
内存寻址速度:ns,内存读写速度:20GB/s
页大小4K,扇区512,避免页索引太大
磁盘是消耗品会有损坏
降低上下文切换的频率和内存拷贝次数
read,write应用程序通过pagecache磁盘预读64K,基于局部性原理,预读的内容会多次读写
mmap,文件传输,业务逻辑处理,明确不需要经常读,可能就是一次性读
sendfile,静态文件传输,明确就是用于传输,不会进行业务逻辑处理,nginx sendfile 前端静态文件
```
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/5e899279.png)
#拓扑
##user buffer
为了传输 320MB 的文件，在用户缓冲区分配了 32KB 的内存，把文件分成 1 万份传送，然而，这 32KB 是怎么来的？为什么不是 32MB 或者 32 字节呢？
这是因为，在没有零拷贝的情况下，我们希望内存的利用率最高。如果用户缓冲区过大，它就无法一次性把消息全拷贝给 socket 缓冲区；
如果用户缓冲区过小，则会导致过多的 read/write 系统调用。
##read/write vs mmap vs sendfile
###read/write(多次读写)
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/1f389e6b.png)

###mmap(基本一次)
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/cf5d4017.png)
将磁盘文件映射到内存, 用户通过修改内存就能修改磁盘文件
直接利用操作系统的Page来实现磁盘文件到物理内存的直接映射。完成映射之后你对物理内存的 操作会被同步到硬盘上(操作系统在适当的时候
通过mmap，进程像读写硬盘一样读写内存(当然是虚拟机内存)。使用这种方式可以获取很大的I/O提升，省去 了用户空间到内核空间复制的开销

###sendfile(仅仅传输,网卡,直接内存访问技术)
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/949b48f3.png)
如果网卡支持 SG-DMA（The Scatter-Gather Direct Memory Access）技术，还可以再去除 Socket 缓冲区的拷贝，这样一共只有 2 次内存拷贝
##⻚缓存Page cache
就是把磁盘中的数据缓存到内存中，把对磁盘的访问变为对内存的访问
磁盘高速缓存
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/270e01c1.png)
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/31cf81b7.png)
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/37ee04a3.png)
###IO 合并与预读
##socket buffer
那用户缓冲区为什么不与 socket 缓冲区大小一致呢？这是因为，socket 缓冲区的可用空间是动态变化的，它既用于 TCP 滑动窗口，也用于应用缓冲区，还受到整个系统内存的影响
##protocol engine
##上下文切换(内核->用户态,用户态->内核,ns*10000)
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/1f389e6b.png)
read/write情况下,上下文切换的成本并不小，虽然一次切换仅消耗几十纳秒到几微秒，但高并发服务会放大这类时间的消耗。
##CPU拷贝(消耗cpu资源)
这个方案做了 4 万次内存拷贝，对 320MB 文件拷贝的字节数也翻了 4 倍，到了 1280MB。很显然，过多的内存拷贝无谓地消耗了 CPU 资源，降低了系统的并发处理能力。
##DMA拷贝
##零拷贝
##直接IO & 异步 IO
高并发场景处理大文件时，应当使用异步 IO 和直接 IO 来替换零拷贝技术
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/b5016102.png)
![](.z_操作系统_01_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/e4eab2dd.png)

#磁盘io关联概念
##物理内存和虚拟内存
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/393ba7be.png)
虚拟内存为每个进程提供了一个一致的、私有的地址空间，它让每个进程产生了一种自己在独享主存的错觉
###进程虚拟内存页表
逻辑地址空间分为若干页,由页表管理
```asp
当进程执行一个程序时，需要先从先内存中读取该进程的指令，然后执行，获取指令时用到的就是虚拟地址。
这个虚拟地址是程序链接时确定的（内核加载并初始化进程时会调整动态库的地址范围）。
为了获取到实际的数据，CPU 需要将虚拟地址转换成物理地址，CPU 转换地址时需要用到进程的页表（Page Table），
而页表（Page Table）里面的数据由操作系统维护。

其中页表（Page Table）可以简单的理解为单个内存映射（Memory Mapping）的链表（当然实际结构很复杂），
里面的每个内存映射（Memory Mapping）都将一块虚拟地址映射到一个特定的地址空间（物理内存或者磁盘存储空间）。
每个进程拥有自己的页表（Page Table），和其它进程的页表（Page Table）没有关系。
```
###物理内存页表
物理内存空间分为若干页框（也叫作块
##进程的用户空间和进程内核空间
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/7c139846.png)
```asp
进程私有的虚拟内存：每个进程都有单独的内核栈、页表、task 结构以及 mem_map 结构等。
进程共享的虚拟内存：属于所有进程共享的内存区域，包括物理存储器、内核数据和内核代码区域。
```

##cpu io中断技术
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/5f953b0a.png)
##DMA io技术
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/a6a1a182.png)

##内存区域映射技术

#磁盘io过程
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/f0ffbb8d.png)
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/87dbf29c.png)
##2次cpu拷贝
CPU拷贝：由 CPU 直接处理数据的传送，数据拷贝时会一直占用 CPU 的资源
CPU将数据从内核内存拷贝到用户进程
##2 次 DMA 拷贝
由 CPU 向DMA磁盘控制器下达指令，让 DMA 控制器来处理数据的传送，数据传送完毕再把信息反馈给 CPU，从而减轻了 CPU 资源的占有率
DMA将数据从磁盘拷贝到内核内存
##4次上下文切换
当用户程序向内核发起系统调用时，CPU 将用户进程从用户态切换到内核态；当系统调用返回时，CPU 将用户进程从内核态切换回用户态。
##传统读
```asp
1次CPU拷贝
1次DMA拷贝
2次上下文切换
```
##传统写
```asp
1次CPU拷贝
1次DMA拷贝
2次上下文切换
```
#零拷贝技术
##用户态直接IO(mysql)
```asp
应用程序可以直接访问硬件存储，操作系统内核只是辅助数据传输。这种方式依旧存在用户空间和内核空间的上下文切换，硬件上的数据直接拷贝至了用户空间，
不经过内核空间。因此，直接 I/O 不存在内核空间缓冲区和用户空间缓冲区之间的数据拷贝
```
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/17e383f6.png)
如果你是觉得自己有用户空间的cahce，那么你一定是对于操作特殊的文件拥有自己的策略，于是在open系统调用中提供了O_DIRECT标志，
这标志表明这个文件打开后不再使用内核的cache，而是使用自己的cache
##减少数据拷贝次数
```asp
在数据传输过程中，避免数据在用户空间缓冲区和系统内核空间缓冲区之间的CPU拷贝，以及数据在系统内核空间内的CPU拷贝，这也是当前主流零拷贝技术的实现思路
```
###mmap + write
减少了 CPU 拷贝的次数
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/2d1c7c69.png)
###sendfile(使用page cache)
不仅减少了 CPU 拷贝的次数，还减少了上下文切换的次数
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/437fc2e9.png)
```asp
整个拷贝过程会发生 2 次上下文切换，1 次 CPU 拷贝和 2 次 DMA 拷贝
```
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射mmap_顺序读写_直接IO(direct_io)_零拷贝sendfile_预分配_避免swap写_用户进程缓存_内核缓存_磁盘故障_images/1a8172da.png)
[](https://spongecaptain.cool/SimpleClearFileIO/2.%20DMA%20%E4%B8%8E%E9%9B%B6%E6%8B%B7%E8%B4%9D%E6%8A%80%E6%9C%AF.html)
[](https://github.com/Spongecaptain/SimpleClearFileIO)
数据在内核的pagecache，通过sendfile直接放入socket的sendqueue发出，没有拷贝到程序空间
sendfile是内核方法，pagecache是内核内存，sendqueue也是内核内存
```asp
1.从磁盘文件系统读到的文件，都是以page cache缓存到内存吗？是
2.除了socket  buffer，磁盘读到的资源也会有buffer吗？都是page cache
3.java文件写buffer，也是先写到page cache吧，然后操作系统flush?是
```
###sendfile + DMA gather copy
```asp
为 DMA 拷贝引入了 gather 操作。它将内核空间（kernel space）的读缓冲区（read buffer）中对应的数据描述信息（内存地址、地址偏移量）记录
到相应的网络缓冲区（ socket buffer）中，由 DMA 根据内存地址、地址偏移量将数据批量地从读缓冲区（read buffer）拷贝到网卡设备中，
这样就省去了内核空间中仅剩的 1 次 CPU 拷贝操作
```
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/f2dea5c0.png)
基于 sendfile + DMA gather copy 系统调用的零拷贝方式，整个拷贝过程会发生 2 次上下文切换、0 次 CPU 拷贝以及 2 次 DMA 拷贝
sendfile + DMA gather copy 拷贝方式同样存在用户程序不能对数据进行修改的问题，而且本身需要硬件的支持，它只适用于将数据从文件拷贝到 socket 套接字上的传输过程
```asp
相较传统read/write方式，2.1版本内核引进的sendfile已经减少了内核缓冲区到user缓冲区，再由user缓冲区到socket相关缓冲区的文件copy，
而在内核版本2.4之后，文件描述符结果被改变，sendfile实现了更简单的方式，系统调用方式仍然一样，细节与2.1版本的不同之处在于，
当文件数据被复制到内核缓冲区时，不再将所有数据copy到socket相关的缓冲区，而是仅仅将记录数据位置和长度相关的数据保存到socket相关的缓存，
而实际数据将由DMA模块直接发送到协议引擎，再次减少了一次copy操作
```
###splice
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/dc63caf7.png)
##写时复制技术
```asp
写时复制指的是当多个进程共享同一块数据时，如果其中一个进程需要对这份数据进行修改，那么将其拷贝到自己的进程地址空间中，
如果只是数据读取操作则不需要进行拷贝操作。
```
#java零拷贝
##MappedByteBuffer(内存映射mmap)
内部使用DirectByteBuffer
##FileChannel sendfile
#netty零拷贝
内核态拷贝优化
```
Netty 通过 DefaultFileRegion 类对 java.nio.channels.FileChannel 的 tranferTo() 方法进行包装，在文件传输时可以将文件缓冲区的数据直接发送到目的通道（Channel）
```
应用用户态拷贝优化
```asp
ByteBuf 可以通过 wrap 操作把字节数组、ByteBuf、ByteBuffer 包装成一个 ByteBuf 对象, 进而避免了拷贝操作
ByteBuf 支持 slice 操作, 因此可以将 ByteBuf 分解为多个共享同一个存储区域的 ByteBuf，避免了内存的拷贝
Netty 提供了 CompositeByteBuf 类，它可以将多个 ByteBuf 合并为一个逻辑上的 ByteBuf，避免了各个 ByteBuf 之间的拷贝
```

#案例
##rocketMQ VS kafka
```asp
RocketMQ 选择了 mmap + write 这种零拷贝方式，适用于业务级消息这种小块文件的数据持久化和传输
而 Kafka 采用的是 sendfile 这种零拷贝方式，适用于系统日志消息这种高吞吐量的大块文件的数据持久化和传输。但是值得注意的一点是，
Kafka 的索引文件使用的是 mmap + write 方式，数据文件使用的是 sendfile 方式
```
![](.z_操作系统_磁盘io流程_磁盘读写优化_内存映射_顺序读写_直接IO_零拷贝_预分配_避免swap写_用户进程缓存_内核缓存_images/673c5421.png)

#顺序读vs随机读
节约磁盘寻址时间
[](https://time.geekbang.org/column/article/79368)
[](https://blog.csdn.net/weixin_30537231/article/details/114507875)
#read write vs mmap
mmap建立映射关系开销大,适合大文件io
read write适合小文件
#mmap vs sendfile
[](https://segmentfault.com/q/1010000041019461)
mmap擅长小文件,例如kafka 索引文件10m
sendfile擅长大文件,例如kafka日志文件1G,nginx静态文件

对于RocketMQ来说，因为RocketMQ将所有队列的数据都写入了CommitLog，消费者批量消费时需要读出来进行应用层过滤，
所以就不能利用到sendfile+DMA的零拷贝方式，而只能用mmap
