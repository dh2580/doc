#### 1、TCP、UDP的区别
+ TCP是面向连接的，UDP是面向无连接的，即TCP需要双方三次握手才能建立连接，UDP不需要
+ TCP是面向字节流的，UDP是面向**数据报**的
+ TCP是可靠的有序数据传输，UDP是不可靠的，也不能保证数据的传输顺序
+ TCP适用于消息发送、信号传递等需要可靠性保证的信息系统，UDP适用于直播、在线视频播放等视频流场景

#### 2、TCP协议如何保证可靠传输
+ 通过对每个TCP报文加上序列号，利用滑动窗口来缓存已发送未确认的报文（发送端），ACK累计确认和超时重传机制来保证数据报文传递的可靠性、有序性

#### 3、TCP的粘包/拆包原因及其解决方法是什么
+ 操作系统在发送TCP数据时，底层会有一个缓冲区，如果一次请求发送的数据量比较小，达不到缓冲区的大小，TCP则会将多个请求合并为一个请求进行发送，这就会造成粘包
+ 如果一次请求发送的数据量比较大，超过了缓冲区的大小，TCP就会将其拆分为多次发送，这就会造成拆包

#### 4、Netty粘包/拆包是怎么处理的，有哪些实现
+ 固定数据包长度，在发送数据包时，每个数据包都固定长度，如果发送的数据包不足指定大小，则进行填充空格等特殊字符达到指定长度，具体的实现有`FixedLengthFrameDecoder`

+ 基于换行符，通过数据流中的换行符对数据进行分割，具体实现有`LineBasedFrameDecoder`

+ 基于末尾特殊分隔符，通过指定特殊分隔符作为分割标志，然后基于分隔符对数据流进行分割成单个数据报文，具体实现有`DelimiterBasedFrameDecoder`

+ 消息头部预置消息长度，在数据报文的头部添加长度字段，根据长度字段指定的包长度对接受到的数据进行解码，从而得到消息体数据，具体实现有 `LengthFieldBasedFrameDecoder`与`LengthFieldPrepender`组合使用

+ 自定义粘包器拆包器，通过实现`MessageToByteEncoder`、`ByteToMessageDecoder`来自定义粘包和拆包处理，其中`MessageToByteEncoder`接口将响应数据编码为一个`ByteBuf`对象，`ByteToMessageDecoder`是将接收到的`ByteBuf`数据解码为某个对象数据

#### 5、同步与异步、阻塞与非阻塞的区别？
+ 同步/异步是指操作与操作之间的一个关系，同步是指一系列的操作必须要一个一个去执行（比如B/S架构设计，客户端提交请求 -> 服务端处理 -> 服务端返回 -> 客户端呈现结果），等前一个操作完成后才能执行后面的操作。而异步可以同时执行多个操作，并且通过回调通知的方式通知调用者执行结果

+ 阻塞/非阻塞往往是针对一个线程内而言的，即当前线程调用了一个函数后，在没有得到调用结果之前，是否可以立即返回

+ 此外，针对于同步IO/异步IO来说，最大的区别就是数据拷贝时应用线程是否阻塞，异步IO的数据拷贝由内核拷贝完成后通知应用线程，同步IO的应用线程必须等待内核拷贝完数据才能执行下一步操作

+ 最后，也可以理解为阻塞是使用同步机制的结果（单个线程执行完所有操作，或者所有的线程必须顺序执行，比如同步IO中的应用线程必须等待内核线程执行完才能继续往下执行），非阻塞是使用异步机制的结果（多个线程分别执行各自分配的操作，之间通过回调通知来相互告知彼此的执行结果，比如异步IO中，内核线程执行完数据拷贝后回调通知应用线程）

#### 6、五种网络IO模型
+ 阻塞IO：应用调用一个IO函数，导致应用被阻塞，直到数据被准备好，数据拷贝完成，IO函数返回成功指示

+ 非阻塞IO：应用反复调用IO函数，若无数据准备好，立马返回，如有数据准备好，则等待数据拷贝完成返回，其中，数据拷贝过程中，应用是阻塞的

+ IO多路复用：同时监听多个IO通道，通过调用SELECT、EPOLL等轮询函数，阻塞应用线程，直到监听的IO通道中有数据就绪，同样数据拷贝过程，应用也是阻塞的

+ 信号驱动IO：对监听的IO通道，安装一个信号处理函数。当有数据准备好时，生成一个信号发送给应用，通知应用回调IO函数去处理数据，相比IO多路复用，免去了等待有数据就绪之前的阻塞等待，但是对于数据拷贝过程来说，应用同样也是阻塞的。

+ 异步IO：通过系统调用告诉内核在IO通道有数据可读可写时，执行某个操作，并且让内核在执行完整个操作后通知应用，最主要的特征是内核在进行数据拷贝时，应用不阻塞，数据拷贝完成后才通知应用


#### 7、select、poll、epoll的机制及其区别
+ `select` 通过将应用进程逐个添加到需要监视的`socket fd`的等待队列上，当监视的`socket fd`有一个或者多个有数据就绪时，通过中断程序唤醒应用进程，将应用进程从所有的等待队列中移除，应用进程遍历其监视的`socket fd`列表，得到就绪的`socket`集合

+ `poll` 原理与`select`很相似，但是`socket fd` 集合的数据结构采用了链式结构，没有最大连接数的限制，但同样存在轮询性能问题

+ `epoll` 通过引入`eventpoll`对象，将`eventpoll`对象添加到所有需要监视的`socket fd` 的等待队列中，同时将应用进程添加到`eventpoll`对象的等待队列中，当监视的`socket fd`有就绪数据时，将该`socket fd`添加到`eventpoll`对象的就绪列表中，唤醒`eventpoll`对象的等待队列中的应用进程，唤醒后的应用进程通过只读取就绪列表就可获取就绪的`socket fd`。其中，`eventpoll`中的就绪列表采用了双向链表结构，对于管理监视的`socket fd`集合则采用了红黑树结构。

#### 8、Netty跟Java NIO有什么不同，为什么不直接使用JDK NIO类库
+ 相比`JDK`原生的`NIO`库API，使用`Netty`操作实现起来更为简洁，以及通过`ChannelPipeline`上添加`ChannelHandler`可以很轻松对数据报文进行处理

+ 对于网络闪断、半包读写、异常码流等问题，Netty都有比较成熟的解决方案

+ 针对内存管理，`Netty`提供了内存池来复用内存，减少了向系统申请/释放内存的时间消耗，以及通过对象池避免了`ByteBuf`等对象的重复创建与回收

+ 针对`JDK7`以前版本，包括`JDK7`存在`epoll`空轮询`Bug`，`Netty`中做了对应的解决容错处理（当空轮询超过一定的阈值时，默认是512次，可通过设置`io.netty.selectorAutoRebuildThreshold`参数改变，重新创建一个新的`Select`组件，置换掉原有的`Select`组件）

+ `Netty`中对于原生`JDK NIO`中的一些组件做了优化，比如`ByteBuf`在使用时，就不需要像原生JDK NIO中那样执行`flip()`、`rewind()`操作

#### 9、Netty组件有哪些，分别有什么关联
+ `ByteBuf` : 字节数据缓冲区组件，负责存放数据报文，可以在堆内存、本地直接内存上创建该对象，是Netty中的基础组件，其他组件的逻辑处理都离不开该组件
    
+ `Channel`: 提供了基础的网络IO操作，包括网络数据读写、网络连接、网络链路关闭等，以及其他的一些辅助功能

+ `EventLoop`、`EventLoopGroup`：`EventLoopGroup`是一个IO线程组，包含多个`EventLoop`，每个`EventLoop`代表一个IO线程，负责处理网络IO读写事件。

+ `EventLoop`将`Channel`注册到轮询组件`Selector`上，并且将`Channel`绑定到该`EventLoop`上，`EventLoop`负责绑定到其上面的`Channel`的网络IO读写事件处理，`EventLoop`和`Channel`之间是一对多关系，一个`EventLoop`可以分配多个`Channel`，但是一个`Channel`不能绑定到多个`EventLoop`上

#### 10、Netty的执行流程
+ 服务端建立boss线程池、worker线程池，配置启动相关参数，关联一个childHandler并在childHandler里面，设置ChannelPipeline上的相关ChannelHandler组件，这些组件类似于`Servelet`中的`Filter`，用于对接受到的或者即将对外写出的数据报文进行处理，最后监听指定端口启动服务。客户端向服务端发起连接后，服务端boss线程池负责接收客户端发来的连接，将客户端的连接注册到轮询组件Selector上面。Selector不停地轮询注册到其上面的channel，检测是否有就绪的channel，如果有，则触发IO事件。事件通过ChannelPipeline进行传递到对应的ChannelHandler，ChannelHandler对IO事件进行处理，读取数据报文，业务逻辑处理，生成响应数据，将响应数据写入到SocketChannel，此过程由worker线程池执行。对于数据出站，同样要经过一系列对应的ChannelHandler处理后才会通过网络发送到客户端。

#### 11、Netty的线程模型是怎么样的
+ 采用`Reactor`线程模型，通过配置启动参数，可以支持`Reactor`单线程模型、`Reactor`多线程模型、主从`Reactor`多线程模型。Netty服务端在启动时，创建两个独立的线程池（`NioEventLoopGroup`），一个线程池负责接受客户端的`TCP`连接，另一个线程池负责处理`I/O`相关的读写操作。此外，为了避免由于多个IO线程并发操作同一个`channel`导致并发问题，或者引入同步机制导致处理更加复杂，采用了每个`channel`绑定到一个特定的线程上，此后该线程负责此`channel`的读写事件处理。线程与绑定的`channel`之间是一对多关系，即一个线程上可以绑定多个`channel`，但是一个`channel`不能绑定到多个线程上

#### 12、Netty的零拷贝提体现在哪里，与操作系统上的有什么区别
+ 操作系统层面的零拷贝是指避免在用户态（`User-Space`）、内核态（`Kernel-Space`）之间来回拷贝数据，例如Linux中提供的`mmap`（内存映射）系统调用，将一段用户空间映射到内核空间，用户对这段内存空间的修改可以直接反映到内核空间，同样内核空间对这段内存空间的修改也可以直接反映到用户空间

+ Netty中的零拷贝不同于操作系统的零拷贝，Netty的零拷贝是完全在用户态层面的，与其说是零拷贝，更不如说是优化数据操作的一种手段，具体体现在如下

+ Netty提供了`CompositeByteBuf`类，将多个ByteBuf合并为一个`逻辑上`的ByteBuf，避免了在各个ByteBuf之间的拷贝

+ 通过ByteBuf的`wrap`操作，可以将`byte[]数组`、`ByteBuf`、`ByteBuffer`包装成一个`Netty ByteBuf`对象，避免了拷贝操作

+ ByteBuf还支持`slice`操作，可以ByteBuf分解为多个共享同一存储的ByteBuf，避免了内存拷贝，类似于`ArrayList`的`subList(...)`方法

#### 13、Netty的内存池实现原理
+ Netty内存由多个`Arena`（内存区域）组成，每个内存区域由多个`Chunk`（块）组成，每个`Chunk`又由多个`Page`（页）组成，每个`Page`由多个`SubPage`（子页）组成。

+ 每个`PoolArena`中有多个表示不同内存使用率的`PoolChunkList`，而每个`PoolChunk`中，默认有2048个Page。

+ 在PoolChunk中，Page的大小是固定，默认8K，Page被构建成一颗二叉树，每个Page的父节点能分配当前节点能分配空间的double空间。当一个节点代表的内存区域被分配出去后，这个节点会被标记为已分配，自这个节点以下的所有节点在后续的内存分配请求中都会被忽略。
+ 对于每个Page的内部，由多个SubPage组成，每个SubPage大小都是相同的，但是SubPage的大小取决于第一次在该Page内申请的内存大小。如果Page大小为8K，第一在Page内申请内存大小为1K，则自此以后该Page只能分配1K大小的内存，超过1K则需在新的Page中分配。

#### 14、Netty的对象池是怎么实现的
+ 将每一个缓存的对象包装成为一个`DefaultHandle`类对象，利用`Stack`对象存储回收的对象，以及获取对象，回收和获取分别对应于`push()`、`pop()`操作。`Stack`对象包含了一个本地的对象数组，和一个`WeakOrderQueue`对象组成的链表，其中本地对象数组存储了本线程回收的对象，`WeakOrderQueue`链表中的每一个节点存储了其他的一个线程回收到该`Stack`中的对象。当取对象时，优先从本地对象数组中取，如果本地对象数组中取不到对象，则从`WeakOrderQueue`链表中优先获取head `WeakOrderQueue`的head `Link`中的回收对象，置换到本地对象数组。(每个`WeakOrderQueue`都对应着一个`Link`类型的链表 ，每个`Link`节点都包含一个对象数组，用于本地对象数组为空时，从`WeakOrderQueue`中获取`DefaultHandle`，迁移到`Stack`中的本地对象数组。)。

+ 缓存的对象大多数是`ByteBuf`对象，比如`PooledHeapByteBuf`、`PooledDirectByteBuf`

#### 15、`ChannelPromise` 与 `ChannelFuture`的区别

+ `ChannelFuture` 用于获取IO操作的结果，以及注册结果监听器

+ `ChannelPromise` 不仅可以用于获取IO操作的结果、注册结果监听器，而且还可以设置IO操作的结果，相当于可写的`ChannelFuture`

#### 引用

+ [Netty5 线程引用关系](https://www.cnblogs.com/tiancai/p/9959572.html)