
# 引言
GFS是谷歌2003年提出的一个文件系统。虽然GFS比较古老，但是后来的HDFS，是受到了GFS的启发，是GFS的一种开源实现。因此熟悉与理解GFS的设计原理，会对理解整个hadoop生态系统有更好的帮助。

# 系统目标
GFS(Goole File System)是为了满足谷歌生产过程中存储的海量数据的生成和处理的需求而设计的一个分布式文件系统。这个文件系统基于大量廉价的硬件之上。

# 指导思想
因此，GFS设计的主要指导思想为：

- 1. 组件故障应当被视作日常而不是异常(基于大量廉价硬件)

- 2. 文件的大小通常比较大(日志文件，几个G很常见)

- 3. 写文件的方式以追加为主，很少会覆盖(日志文件，一般都是追加文件)

- 4. 客户端的使用和文件系统是一起设计的，比较灵活(没必要完全按照POSIX的标准来，客户端只要在使用过程中稍微注意一点，就能给服务端减少很多复杂的设计。后面会提到GFS一个独有的atomic append的操作，这个操作可能会导致不一致性，但是不影响客户端的使用)

# 基于假设
从上述指导思想中，衍生出了系统设计过程中的假设

- 1. 整个系统基于许多廉价的硬件，因此容易坏，系统能够在某个硬件失效的时候快速恢复；

- 2. 系统存在许多大文件；

- 3. 读文件的方式以顺序读为主，支持少量的随机读；

- 4. 写文件的方式以顺序写为主，而且写完很少会修改，随机写文件也会被支持，但是可能效率很差；

- 5. 支持多个客户端同时写一个文件；

- 6. 宽带的吞吐量高比延迟低更重要，写日志通常是比较大的数据，因此系统能够稳定的处理大量的数据比快速响应请求更为重要。

# 整体架构

可以先看看论文里面的架构图。

![](https://upload-images.jianshu.io/upload_images/4309024-3e9b3ed1e3bf65c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

# GFS架构

GFS主要包括master和chunk server两部分组成。一个文件被打散成了多个chunk，每个chunk会在多个chunk server上保存作为冗余备份。客户端向master请求某个某个文件的某个chunk的数据，master返回这个chunk所属的chunk server。客户端再请求对应的chunk server操作数据。

其中，master机器只有一台，在内存中存储：

- 1. File namespace: 可以理解为文件目录结构；

- 2. 每个文件所包含的chunk；

- 3. 每一个chunk存放在哪些chunk server上面(不在master进行持久化，master启动时询问所有的chunk server，chunk server加入时也会主动注册)。

其中，对1(如新建文件、删除文件等)和2的修改(文件添加一个新的chunk等)，会先写operation log(master和shadow master上都要写)，全部写完了之后再操作内存。

对于3来说，master并不会持久化存储这些信息，而是在master启动时询问所有的chunk server有哪些chunk，chunk server加入时也会主动告诉master。

chunk server机器会有很多台，每台机器上都存储了不同的chunk，每一个chunk被设定为64M的大小（为了满足大文件的需求）。

看到这里，会存在几个疑问：

- 这种数据存储形式有什么好处？

这样的设计巧妙的解耦了元数据和数据。对于元数据，只有master知道。因此，master必须好好维护这些信息，防止数据丢失或者错误，因此写operation log就变得非常重要。而对于数据本身，chunk server更加了解自身存储的情况，因此chunk的信息，chunk server说了算。这样避免了master中持久化一份数据分布的信息而带来的与实际情况不一致的情况(另外设计一套同步机制会使系统变得复杂)。

- master机器只有一台，怎么避免单点故障？

为防止单点故障，肯定要准备备机。一般会存在2台或者以上的shadow master。当master在本地写operation log的时候，也会让shadow master写一份。只有在全部都写成功的时候，操作才会继续。如果master的硬盘挂了，那么任意一台shadow master都能够挺身而出充当新的master。

- master机器的内存大小会不会成为系统存储容量的瓶颈呢？

首先，一个chunk被设置成了64M，大大减少了一个文件所占用的chunk数量，因此，master中存储的数量也会大大减少。

其次，由于系统存储的是大文件，因此每一个文件，除了最后一个chunk以外，其他的chunk基本上都是存满的。

最后，加内存也不是什么难事。

一般64个bytes就可以存一个64M的chunk信息，因此大约可以用(内存大小)*(2^6)估算系统的容量，至少可以达到P级别。

operation log是记下来了，但是重启的时候会不会因为日志回放的时间太久而导致很慢？

当operation log达到一定大小时，系统会切换到一个新的日志。老的日志则会被回放，并存储在硬盘上，这个操作被称作checkpoint。当master重启时，直接加载checkpoint生成的内容，并回放新的日志，就可以完成了。

# 交互流程
租约：对于每个chunk都有多个副本。当客户端要修改数据的时候，master通过使用租约来保证多个副本之间的一致性。master从所有副本中选出一个chunk server发放租约，得到租约的副本，称为primary，其余的就是secondary。租约时长60s。primary将同一个chunk的操作进行序列化。通过心跳包可以续租。master server可以主动收回租约。
写操作的控制流和数据流：
![](https://upload-images.jianshu.io/upload_images/4309024-c33ad248b9e11d86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/482)


客户端修改数据流程流程

- 1. client请求master某个chunk;

- 2. master告诉client拥有这个chunk的primary和secondary分别是谁;

- 3. client将自己的操作数据传给其中某个chunk server，chunk server在收到数据之后，立刻推给最近的另外一台没有接受过数据的chunk server，直到所有chunk server都有这个数据。这种数据分发方式有助于充分利用系统内的带宽;

- 4. client告诉primary进行操作;

- 5. primary告诉其他secondary进行操作;

- 6. 如果所有secondary操作成功，则整个操作成功;

- 7. 如果操作失败，由primary告诉clinet，并进行重试。

# 对外提供API

- create
只在master上操作。举个例子，如要创建/d1/d2/.../dn/leaf文件，则需要获得/d1的读锁，/d1/d2的读锁，...，/d1/d2/.../dn的读锁，/d1/d2/.../dn/leaf的写锁。取锁成功后，写operation log，完成后在内存生成该条目，操作成功。

- delete
只在master上操作。举个例子，如要删除/d1/d2/.../dn/leaf文件，则需要获得/d1的读锁，/d1/d2的读锁，...，/d1/d2/.../dn的读锁，/d1/d2/.../dn/leaf的写锁。取锁成功后，写operation log，完成后从内存删除该条目，操作成功。文件对应的chunk不实时删除，将在garbage collection阶段异步处理。

- open、close、read、write
文章并未详细提及。

- snapshot
复制某个文件或者目录。当收到snapshot请求时，master立即收回所有该文件已经发出的租约。当所有租约被收回或者过期之后，写operation log，完成后操作内存复制文件或目录，并且将文件的所有chunk指向与原文件一致。此时，chunk的引用计数为2。当master收到该文件的某个chunk，比如C，的写请求之后，master意识到C的引用计数是2，则要求所有拥有C的chunk server本地复制一份C'，之后的写请求就与正常情况下一致了。

- record append
最常使用的一种写入方式，该操作是原子操作，多个客户端可以同时向primary请求执行这个命令，这些命令会被放在队列当中，由primary一个个来完成追加的操作(多生产者/单消费者模型)。

如果primary发现追加的内容：

- a.大于该chunk剩余的空间，就会将chunk剩余的部分填充到最大的64M，并通知secondary也进行同样的操作，并通知客户端需要使用新的chunk来操作。客户端之后重新发起请求，后面的流程与b一致。

- b.大小够用，则写入，并通知secondary同样写入，并将结果返回给客户端。

b执行过程中有两种可能情况：

- primary执行失败。则直接返回失败重试。

- primary执行成功，但secondary中有一个或者多个返回失败。返回客户端失败让其重试，但primary和secondary中间成功的不回滚之前的操作。因此，当客户端重试的时候，primary中会产生两条相同的记录。几个chunk server之间可能会发生不一致的情况。

这种不一致的情况允许存在，也就是说GFS放宽了server端一致性的要求。客户端需要对这种情况有所准备，需要有能够过滤重复数据的能力。

# 其他内部机制
- 副本的放置
- 为了防止同一个机架上的server一起挂掉，副本尽量会跨机架放置。

# chunk的创建

在三种场景下会发生：

- 1. 新chunk的创建；

- 2. chunk的副本数低于阈值，需要再复制一些；

- 3. chunk副本的分布不是很好，需要优化。

总而言之还是要尽量跨机架。

# 垃圾回收

- 1. master会定时检测没有被文件引用的chunk并删除。

- 2. 在master和chunk server的心跳包中，chunk server会报告目前拥有的chunk，master跟自己的对比之后返回可以删除的chunk，然后chunk server进行删除操作。

# 陈旧副本的侦测
master发放租约的时候，会带上一个版本号给所有的副本，如果某个副本机器挂了，没有拿到这个版本号，那么在该副本重启的过程中，master会发现版本号不一致，则会让该副本失效。

# 总结
GFS 适合于大数据量读写，大多是追加操作的场景。后面MapReduce会进行更一步的改进。

# Reference

- [Google File System中文版](http://blog.bizcloudsoft.com/wp-content/uploads/Google-File-System%E4%B8%AD%E6%96%87%E7%89%88_1.0.pdf)
- [Google File System](http://web.eecs.umich.edu/~mosharaf/Readings/GFS.pdf)
- 以及开设这门课程老师的PPT

