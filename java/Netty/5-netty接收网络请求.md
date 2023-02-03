# Netty接收网络请求

之前在服务端启动时说过基本的事件处理逻辑，下面主要关注服务端接收从客户端过来的请求，主要分为 `accept`和 `read`事件的处理。

参考: [Netty是如何高效接收网络连接的](https://heapdump.cn/article/4130911)

```java
// NioEventLoop#processSelectedKey
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
}
// AbstractNioMessageChannel#read 或者
// AbstractNioByteChannel#read
```

这里首先强调下，在读取客户端的连接请求（`OP_ACCEPT`事件）时，相关数据长度是固定的，所以使用的是 `AbstractNioMessageChannel#read`，而后面在读取客户端数据（`OP_READ`事件）时，由于数据长度不是固定的，所以使用的是 `AbstractNioByteChannel#read`。

# 1. RecvByteBufAllocator简介

`RecvByteBufAllocator`是一个数据 `Buffer`分配器，用户分配容量大小合适的缓存，避免浪费。

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
public DefaultChannelConfig(Channel channel) {
    this(channel, new AdaptiveRecvByteBufAllocator());
}
```

以上就是其创建过程。

> 对于服务端 `NioServerSocketChannel`来说，它上边的 `IO`数据就是客户端的连接，它的长度和类型都是固定的，所以在接收客户端连接的时候并不需要这样的一个 `ByteBuffer`来接收，我们会将接收到的客户端连接存放在 `List<Object> readBuf`集合中

> 对于客户端 `NioSocketChannel`来说，它上边的 `IO`数据时客户端发送来的网络数据，长度是不定的，所以才会需要这样一个可以根据每次 `IO`数据的大小来自适应动态调整容量的 `ByteBuffer`来接收。

## 1.1 获取

```java
// AbstractNioMessageChannel#read
final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();

// AbstractChannel.AbstractUnsafe.java
public RecvByteBufAllocator.Handle recvBufAllocHandle() {
    if (recvHandle == null) {
        recvHandle = config().getRecvByteBufAllocator().newHandle();
    }
    return recvHandle;
}

// AdaptiveRecvByteBufAllocator
public Handle newHandle() {
    return new HandleImpl(minIndex, maxIndex, initial);
}
private final class HandleImpl extends MaxMessageHandle {
    private final int minIndex;
    private final int maxIndex;
    private int index;
    //预计下一次分配buffer的容量，初始：2048
    private int nextReceiveBufferSize;
    private boolean decreaseNow;
    HandleImpl(int minIndex, int maxIndex, int initial) {
        this.minIndex = minIndex;
        this.maxIndex = maxIndex;
        index = getSizeTableIndex(initial);
        nextReceiveBufferSize = SIZE_TABLE[index];
    }
}
public abstract class MaxMessageHandle implements ExtendedHandle {
    private ChannelConfig config;
    // 每次事件轮训时，最多读取（默认16）的最大次数
    // 可在启动配置类ServerBootstrap中通过ChannelOption.MAX_MESSAGES_PER_READ选项设置
    private int maxMessagePerRead;
    // 本次事件轮训总共读取的message数，这里指的是接收连接的数量
    private int totalMessages;
    // 本次事件轮训总共读取的字节数
    private int totalBytesRead;
    // 表示本次read loop 尝试读取多少字节，byteBuffer剩余可写的字节数
    private int attemptedBytesRead;
    // 本次read loop读取到的字节数
    private int lastBytesRead;
}
```

* `maxMessagePerRead`: 用于控制每次事件轮训最大次数，默认16次，可以在 `ServerBootstratp`中通过 `ChannelOption.MAX_MESSAGES_PER_READ`进行设置
* `totalMessages:` 用于统计本次轮训中总共接收的连接个数，针对 `accept`事件的
* `totalBytesRead`: 用于统计本次轮训中总共收到客户端的数据大小，读取时间的处理是在从 `Reactor`上面完成的，而处理 `accept`事件是在主 `Reactor`上面，所以在主 `Reactor`上此字段用不到。此字段回在每次读取完 `NioSocketChannel`数据后增加
  ```java
  public void lastBytesRead(int bytes) {
      lastBytesRead = bytes;
      if (bytes > 0) {
          totalBytesRead += bytes;
      }
  }
  ```

## 1.2 重置

每次使用之前都需要将相关统计数据重置

```java
public void reset(ChannelConfig config) {
    this.config = config;
    //默认每次最多读取16次
    maxMessagePerRead = maxMessagesPerRead();
    totalMessages = totalBytesRead = 0;
}
```

## 1.3 轮训次数判断

```java
// AbstractNioMessageChannel -> DefaultMaxMessagesRecvByteBufAllocator
public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
    return config.isAutoRead() &&
        (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
        totalMessages < maxMessagePerRead && (ignoreBytesRead || totalBytesRead > 0);
}
```

这里要明确的一点就是，`maxMessagePerRead`是针对 `NioServerSocketChannel`的，而 `totalByteRead`是针对 `NioSocketChannel`的。主要关注后面两个判断

* `totalMessages < maxMessagePerRead`： 针对服务端的 `accept`事件[`NioServerSocketChannel`]，处于主 `Reactor`, 判断客户端的连接数是否超过
* `ignoreBytesRead || totalBytesRead >0`：针对 `read`事件[`NioSocketChanel`],处于从 `Reactor`，判断读取的字节数是否达到上线。如果目前在主 `Reactor`上，那么 `ignoreBytesRead=true`，因为主 `Reactor`上不会接收客户端的数据，`totalBytesRead=0`。

# 2. 接收客户端连接-accept事件

```java
// AbstractNioMessageChannel#read ->
// NioServerSocketChannel
protected int doReadMessages(List<Object> buf) throws Exception {
    // 接收客户端的连接，获得客户端socketChannel
    SocketChannel ch = SocketUtils.accept(javaChannel());
    try {
        if (ch != null) {
            // 添加到缓存中，注意这里创建出来的channel被初始化为可读
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        // ...
    }
    return 0;
}
public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    super(parent, ch, SelectionKey.OP_READ);
}
```

可以看到，这里会创建一个 `NioSocketChannel`，而其 `parent`是 `NioServerSocketChannel`。然后添加到缓存中。

> `NioSocketChannel   VS  NioServerSocketChannel`
>
> 1，层次不一样，通过 `NioServerSocketChannel`创建出来的 `NioSocketChannel`，其 `parent`是 `NioServerSocketChaennl`，而客户端 `Reactor`启动时创建的 `NioSocketChannel`其 `parent`是 `null`
>
> 2，客户端 `NioSocketChannel`向从 `Reactor`注册的是 `SelectionKey.OP_READ事件 `，而服务端 `NioServerSocketChannel`向主 `Reactor`注册的是 `SelectionKey.OP_ACCEPT事件`
>
> 3，功能属性不同使得继承结构不同
>
> 客户端 `NioSocketChannel`主要处理的是服务端与客户端的通信，这里涉及到接收客户端发送来的数据，而从 `Reactor`线程从 `NioSocketChannel`中读取的正是网络通信数据单位为 `Byte`。
>
> 服务端 `NioServerSocketChannel`主要负责处理 `OP_ACCEPT`事件，创建用于通信的客户端 `NioSocketChannel`。这时候客户端与服务端还没开始通信，所以主 `Reactor`线程从 `NioServerSocketChannel`的读取对象为 `Message`。这里的 `Message`指的就是底层的 `SocketChannel`客户端连接。

接收完之后会遍历处理，在 `pipeline`中传播连接事件

```java
pipeline.fireChannelRead(readBuf.get(i));
```

此时 `NioServerSocketChannel`的 `pipeline`结构如下

```java
HeadContext --> ServerBootstrapAcceptor --> TailContext
```

`ServerBootstrapAcceptor`主要的作用就是初始化客户端 `NioSocketChannel`，并将客户端 `NioSocketChannel`注册到 `Sub Reactor Group `中，并监听 `OP_READ事件`。

```java
private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {
    private final EventLoopGroup childGroup;
    private final ChannelHandler childHandler;
    private final Entry<ChannelOption<?>, Object>[] childOptions;
    private final Entry<AttributeKey<?>, Object>[] childAttrs;
    @Override
    @SuppressWarnings("unchecked")
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        final Channel child = (Channel) msg;
        //向客户端NioSocketChannel的pipeline中
        //添加在启动配置类ServerBootstrap中配置的ChannelHandler
        child.pipeline().addLast(childHandler);
        //利用配置的属性初始化客户端NioSocketChannel
        setChannelOptions(child, childOptions, logger);
        setAttributes(child, childAttrs);
        try {
            /**
             * 1：在Sub Reactor线程组中选择一个Reactor绑定
             * 2：将客户端SocketChannel注册到绑定的Reactor上
             * 3：SocketChannel注册到sub reactor中的selector上，并监听OP_READ事件
             * */
            childGroup.register(child).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (!future.isSuccess()) {
                        forceClose(child, future.cause());
                    }
                }
            });
        } catch (Throwable t) {
            forceClose(child, t);
        }
    }
}
```

可以看到这里会将所有的从 `Reactor`属性注册到 `NioSocketChannel`上，然后注册 `NioSocketChannel`到从 `Reactor`。

```java
// SingleThreadEventLoop#register -> register ->
// AbstractChannel#register -> register0 -> AbstractNioChannel#doRegister
selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
```

这里相关流程在服务端启动时已梳理，这里不再详述，但是要注意的是这里注册的**事件值是0**，目的只是获取到 `NioSocketChannel`在 `Selector`中的 `SelectionKey`。同时通过 `SelectableChannel#register`方法将 `Netty`自定义的 `NioSocketChannel`（这里的 `this`指针）附着在 `SelectionKey`的 `attechment`属性上，完成 `Netty`自定义 `Channel`与 `JDK NIO Channel`的关系绑定。这样在每次对 `Selector`进行 `IO`就绪事件轮询时，`Netty `都可以从 `JDK NIO Selector`返回的 `SelectionKey`中获取到自定义的 `Channel`对象（这里指的就是 `NioSocketChannel`）。

同时要注意，在 `AbstractChannel#register`方法中传入的 `EventLoop`是从 `Reactor`线程的，但是此时执行线程为主 `Reactor`线程，也就是说监听的任务不会立即执行，而是要等到自定义从 `Reactor#handler`初始化完成之后再执行，这和服务端启动注册的逻辑是类似的。

这里为什么不直接注册 `read`事件呢？因为 `EchoServer`中我们注册的相关自定义 `handler[EchoChannelHandlerEchoServerHandle]`还没有注册完成，需要等其初始号完成，这和之前服务端注册是一样的流程，都涉及到回调。回调完成之后从 `Reactor`的 `pipeline`结构如下

```java
HeadContext --> EchoChannelHandler  --> TailContext
```

然后传播 `channelActive`事件，表明注册完成，在 `HeadContext`中

```java
// AbstractChannel#register -> register0 -> AbstractNioChannel#doRegister
// --> beginRead();
// AbstractNioChannel
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }
    readPending = true;
    // ServerSocketChannel初始化时设置的值为readInterestOp=OP_ACCEPT
    // SocketChannel初始化时设置的值为readInterestOp=OP_READ
    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        // 这里添加OP_ACCEPT/OP_READ到interestOps集合中
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

可以看到，这里才注册 `OP_READ`事件进行监听。

# 3. 接收客户端数据-read事件

客户端发起系统 `IO`调用向服务端发送数据之后，当网络数据到达服务端的网卡并经过内核协议栈的处理，最终数据到达 `Socket`的接收缓冲区之后，`Sub Reactor`轮询到 `NioSocketChannel`上的 `OP_READ事件 `就绪，随后 `Sub Reactor`线程就会从 `JDK Selector`上的阻塞轮询 `API` `selector.select(timeoutMillis)`调用中返回。转而去处理 `NioSocketChannel`上的 `OP_READ事件`。

```java
// AbstractNioByteChannel
public final void read() {
    final ChannelConfig config = config();
    if (shouldBreakReadReady(config)) {
        clearReadPending();
        return;
    }
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    // 自适应ByteBuf分配器 AdaptiveRecvByteBufAllocator ,用于动态调节ByteBuf容量
    // 需要与具体的ByteBuf分配器配合使用 比如这里的PooledByteBufAllocator
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    // PooledByteBufAllocator为Netty中的内存池，用来管理堆外内存DirectByteBuffer
    // 这里的allocHandle就是 MaxMessageHandle
    allocHandle.reset(config);
    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
            // 利用PooledByteBufAllocator分配合适大小的byteBuf 初始大小为2048
            byteBuf = allocHandle.allocate(allocator);
            // 记录本次读取了多少字节的数据
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            // 没有读取到数据或者连接被关闭，那么推出循环
            if (allocHandle.lastBytesRead() <= 0) {
                // nothing was read. release the buffer.
                byteBuf.release();
                byteBuf = null;
                close = allocHandle.lastBytesRead() < 0;
                if (close) {
                    // There is nothing left to read as we received an EOF.
                    readPending = false;
                }
                break;
            }
            // 循环次数+1
            allocHandle.incMessagesRead(1);
            readPending = false;
            // NioSocketChannel中的pipeline触发channelRead事件
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
        } while (allocHandle.continueReading());//默认16次
        // 本次循环结束，根据本次循环总共读取的字节数调整buffer容量
        allocHandle.readComplete();
        // 在NioSocketChannel的pipeline中触发ChannelReadComplete事件，表示一次read事件处理完毕
        // 但这并不表示 客户端发送来的数据已经全部读完，因为如果数据太多的话，
        // 这里只会读取16次，剩下的会等到下次read事件到来后在处理
        pipeline.fireChannelReadComplete();
        if (close) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
```

## 3.1 总体流程逻辑

这里整体逻辑和之前处理客户端连接基本类似，也是先分配适当大小的缓存，然后通过一个循环来处理每个 `ready`的读请求。同时这里同样控制了每次循环的次数（默认16次）以及每个 `channel`读取数据量的大小。这里控制次数的原因很简单，就是一个从 `Reactor`上面会注册多个 `channel`，`Netty`不可能在一个 `channel`上无限制的处理下去，需要进行分摊，同时还有异步任务需要处理。

于是在没循环一次都会记录下次数

```java
allocHandle.incMessagesRead(1);
// MaxMessageHandle
public final void incMessagesRead(int amt) {
    totalMessages += amt;
}
```

以及记录读取的数据量

```java
allocHandle.lastBytesRead(doReadBytes(byteBuf));

// MaxMessageHandle.java
//本次read loop读取到的字节数
private int lastBytesRead;
//整个read loop循环总共读取的字节数
private int totalBytesRead;

public void lastBytesRead(int bytes) {
    lastBytesRead = bytes;
    if (bytes > 0) {
        totalBytesRead += bytes;
    }
}
```

控制读取数据量逻辑

* `lastBytesRead < 0`：表示客户端主动发起了连接关闭流程，`Netty`开始连接关闭处理流程。这个和本文的主旨无关，我们先不用管。
* `lastBytesRead = 0`：表示当前 `NioSocketChannel`上的数据已经全部读取完毕，没有数据可读了。本次 `OP_READ`事件圆满处理完毕，可以开开心心的退出 `read loop`。
* 当 `lastBytesRead > 0`：表示在本次 `read loop`中从 `NioSocketChannel `中读取到了数据，会在 `NioSocketChannel `的 `pipeline `中触发 `ChannelRead `事件。进而在 `pipeline `中负责 `IO `处理的 `ChannelHandler`中响应，处理网络请求。

数据读取到缓存中之后会触发 `pipeline`传递 `channelRead`事件

```java
pipeline.fireChannelRead(byteBuf);
```

其实在 `handler`中还有一个 `channelReadCompleted`，这两个方法有啥区别呢？

之前说过，这里读取每个 `channel`中的数据是有容量限制的，不可能一直在一个 `channel`上面无限制读取，所以这里可能数据还没有读取完毕，那需要等到下次循环过来。每次循环后都需要判断循环次数是不是达到限制，如果达到则退出循环。

然后根据本次循环读取的数据量来调整缓存容量，同时触发 `pipeline`中的 `channelReadCompleted`事件。注意：这里如果某个 `channel`数据量很大，而当前缓存分配又比较小时，可能循环多次后数据还未读取完毕，那这就需要等到下次处理 `read`事件了。当时此时可能在 `channelReadCompleted`方法中已经给客户端返回了相关信息(但其实数据还未完全读取完毕)。

## 3.2 事件处理循环逻辑

```java
final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
```

之前在处理 `OP_ACCEPT`时，这里的 `RecvByteBufAllocator`类型为 `ServerChannelRecvByteBufAllocator`。而目前在处理 `OP_READ`时，这里的 `RecvByteBufAllocator`类型为 `AdaptiveRecvByteBufAllocator`。也是在初始化 `DefaultChannelConfig`时创建的。注意，`AdaptiveRecvByteBufAllocator`并不会真正去分配缓存，而只是负责动态计算缓存的大小，真正负责内存分配的是 `PooledByteBufAllocator`。

```java
private final class HandleImpl extends MaxMessageHandle {
    // 最小容量在扩缩容索引表SIZE_TABE中的index。默认是3。
    private final int minIndex;
    // 最大容量在扩缩容索引表SIZE_TABE中的index。默认是38。
    private final int maxIndex;
    // 当前容量在扩缩容索引表SIZE_TABE中的index。初始是33。
    private int index;
    // 预计下一次分配buffer的容量，初始：2048
    private int nextReceiveBufferSize;
    // 是否需要立即缩容
    private boolean decreaseNow;
}
public abstract class MaxMessageHandle implements ExtendedHandle {
    private ChannelConfig config;
    // 每次事件轮训时，最多读取（默认16）的最大次数
    // 可在启动配置类ServerBootstrap中通过ChannelOption.MAX_MESSAGES_PER_READ选项设置
    private int maxMessagePerRead;
    // 本次事件轮训总共读取的message数，这里指的是接收连接的数量
    private int totalMessages;
    // 本次事件轮训总共读取的字节数
    private int totalBytesRead;
    // 表示本次read loop 尝试读取多少字节，byteBuffer剩余可写的字节数
    private int attemptedBytesRead;
    // 本次read loop读取到的字节数
    private int lastBytesRead;
}
```

在每轮循环开始之前，都会调用 `allocHandle.reset(config)`重置清空上一轮循环的统计指标。

```java
// HandleImpl
public void reset(ChannelConfig config) {
    this.config = config;
    // 默认16次
    maxMessagePerRead = maxMessagesPerRead();
    totalMessages = totalBytesRead = 0;
}
```

在每次开始从 `NioSocketChannel`中读取数据之前，需要利用  `PooledByteBufAllocator `在内存池中为 `ByteBuffer`分配内存，默认初始化大小为 `2048 `，这个容量由 `guess()方法`决定。

```java
// MaxMessageHandle
public ByteBuf allocate(ByteBufAllocator alloc) {
    return alloc.ioBuffer(guess());
}
public int guess() {
    //预计下一次分配buffer的容量，一开始为2048
    return nextReceiveBufferSize;
}
```

每次读取的数据量和循环次数都会统计

```java
allocHandle.lastBytesRead(doReadBytes(byteBuf));
allocHandle.incMessagesRead(1);
```

具体的数据读取其实就是NIO的逻辑，这里省略。

每循环一次都需要判断次数

```java
allocHandle.continueReading()

// MaxMessageHandle
public boolean continueReading() {
    return continueReading(defaultMaybeMoreSupplier);
}
/**
 * 这里主要关注后面两个判断
 * totalMessages < maxMessagePerRead： 针对服务端的accept事件[NioServerSocketChannel]，
 * 处于主Reactor, 判断客户端的连接数是否超过
 * ignoreBytesRead || totalBytesRead > 0：针对read事件[NioSocketChanel],
 * 处于从Reactor，判断读取的字节数是否达到上线。如果目前在主Reactor上，那么ignoreBytesRead=true，
 * 因为主Reactor上不会接收客户端的数据，totalBytesRead=0。这里totalMessages < maxMessagePerRead
 * 表示是否超出最大循环次数
 * 上面是服务端处理OP_ACCEPT的判断，而处理OP_READ时需要注意
 * maybeMoreDataSupplier.get()=ture表示尝试读取数据量和实际读取数据量相等，表明数据可能还未
 * 读取完毕，respectMaybeMoreData默认为true，表示当数据可能还未读取完毕时需要认真对待，
 * 如果设置为false，那么就表示不需要认真对待，继续读取
 */
@Override
public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
    return config.isAutoRead() &&
           (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
           totalMessages < maxMessagePerRead && (ignoreBytesRead || totalBytesRead > 0);
}
private final UncheckedBooleanSupplier defaultMaybeMoreSupplier = new UncheckedBooleanSupplier() {
    public boolean get() {
        // 本次尝试读取的数据量和读取到的数据量相等： 表示ByteBuffer满载而归，说明可能
        // 数据还没有读取完毕。如果不想等，表明数据已经全部读取完毕
        return attemptedBytesRead == lastBytesRead;
    }
};
```

`attemptedBytesRead`-表示当前 `ByteBuffer`预计尝试要写入的字节数

`lastBytesRead`-表示本次循环读取到的数据量

是否继续进行循环需要**同时**满足以下几个条件：

* `totalMessages < maxMessagePerRead` 当前读取次数是否已经超过 `16次`，如果超过，就退出 `do(...)while()`循环。进行下一轮 `OP_READ事件`的轮询。因为每个 `Sub Reactor`管理了多个 `NioSocketChannel`，不能在一个 `NioSocketChannel`上占用太多时间，要将机会均匀地分配给 `Sub Reactor`所管理的所有 `NioSocketChannel`。
* `totalBytesRead > 0` 本次 `OP_READ事件`处理是否读取到了数据，如果已经没有数据可读了，那么就直接退出循环。
* `!respectMaybeMoreData || maybeMoreDataSupplier.get()` 这个条件比较复杂，它其实就是通过 `respectMaybeMoreData`字段来控制 `NioSocketChannel`中可能还有数据可读的情况下该如何处理。

  * `maybeMoreDataSupplier.get()`：`true`表示本次读取从 `NioSocketChannel`中读取数据，`ByteBuffer`满载而归。说明可能 `NioSocketChannel`中还有数据没读完。`fasle`表示 `ByteBuffer`还没有装满，说明 `NioSocketChannel`中已经没有数据可读了。
  * `respectMaybeMoreData = true`表示要对可能还有更多数据进行处理的这种情况要 `respect`认真对待,如果本次循环读取到的数据已经装满 `ByteBuffer`，表示后面可能还有数据，那么就要进行读取。如果 `ByteBuffer`还没装满表示已经没有数据可读了那么就退出循环。
  * `respectMaybeMoreData = false`表示对可能还有更多数据的这种情况不认真对待 `not respect`。不管本次循环读取数据 `ByteBuffer`是否满载而归，都要继续进行读取，直到读取不到数据在退出循环，属于无脑读取。

## 3.3 缓存容量动态调整

### 3.3.1 AdaptiveRecvByteBufAllocator

```java
public class AdaptiveRecvByteBufAllocator extends DefaultMaxMessagesRecvByteBufAllocator {
    // ByteBuffer最小容量，默认为64
    static final int DEFAULT_MINIMUM = 64;
    // Use an initial value that is bigger than the common MTU of 1500
    // ByteBuffer初始化容量
    static final int DEFAULT_INITIAL = 2048;
    // ByteBuffer最大容量
    static final int DEFAULT_MAXIMUM = 65536;
    // 扩容步长
    private static final int INDEX_INCREMENT = 4;
    // 缩容步长
    private static final int INDEX_DECREMENT = 1;
    // RecvBuf分配容量表（扩缩容索引表）按照表中记录的容量大小进行扩缩容
    private static final int[] SIZE_TABLE;

    static {
        // 初始化RecvBuf容量分配表
        List<Integer> sizeTable = new ArrayList<Integer>();
        for (int i = 16; i < 512; i += 16) {
            // 当分配容量小于512时，扩容单位为16递增
            sizeTable.add(i);
        }
        // Suppress a warning since i becomes negative when an integer overflow happens
        for (int i = 512; i > 0; i <<= 1) { // lgtm[java/constant-comparison]
            // 当分配容量大于512时，扩容单位为一倍，最终i会越界变为负值
            sizeTable.add(i);
        }
        SIZE_TABLE = new int[sizeTable.size()];
        for (int i = 0; i < SIZE_TABLE.length; i ++) {
            SIZE_TABLE[i] = sizeTable.get(i);
        }
    }

    /**
     * 二分查找最贴近（第一个大于等于size）给定size容量的索引下标
     */
    private static int getSizeTableIndex(final int size) {
        for (int low = 0, high = SIZE_TABLE.length - 1;;) {
            if (high < low) {
                return low;
            }
            if (high == low) {
                return high;
            }
            // 无符号右移
            int mid = low + high >>> 1;
            int a = SIZE_TABLE[mid];
            int b = SIZE_TABLE[mid + 1];
            if (size > b) {
                low = mid + 1;
            } else if (size < a) {
                high = mid - 1;
            } else if (size == a) {
                return mid;
            } else {
                return mid + 1;
            }
        }
    }
}
```

这里主要关注几个点

* 扩容步长默认为4，缩容步长默认为1
* 根据扩缩容表中的值和步长来进行扩容或者缩容
* 每一次循环结束后根据当前循环读取的总的数据量来决定是扩容还是缩容

扩缩容逻辑比较简单，就是首先根据传入数据值找到其在表中的位置，然后根据步长取到下一个位置上面的值。同时在初始化时可以传入最大和最小的容量值，后面扩容或者缩容不会超出这个限制。

### 3.3.2 PooledByteBufAllocator创建

```java
public class DefaultChannelConfig implements ChannelConfig {
    //PooledByteBufAllocator
    private volatile ByteBufAllocator allocator = ByteBufAllocator.DEFAULT;

    ..........省略......
}
```

具体池化比较复杂，后续再说明。

### 3.3.3 OP_READ标记擦除

首先上面说到，如果循环处理16次，还是没有将某个 `channel`中的数据读取完毕，那么就需要等到下次事件触发再处理，那也就表明，此时循环退出了其 `OP_READ`标记还是在的。但是在有些情况下时需要擦除的

```java
if (close) {
    closeOnRead(pipeline);
}

if (!readPending && !config.isAutoRead()) {
    removeReadOp();
}
```

一种是客户端连接关闭时会将标记擦除，这很好理解，主要是下面的情况。

* `readPending=false`且该 `channel`未开启自动读，则我们取消注册的 `read`事件
* `readPending=true` 代表当前通道正在等待读取事件或者正在处理读取事件，设置为 `false`代表某一次的请求数据读取完毕。
* `readPending=true` 一般都是在我们开始读取完毕之后通过 `ChannelReadComplete` 重新调用 `unsafe`的 `doBeginRead`方法设置的。

# 4. 处理 write 事件

`Netty` 中有两个触发 `write` 事件传播的方法，它们的传播处理逻辑都是一样的，只不过它们在 `pipeline` 中的**传播起点**是不同的。

* `channelHandlerContext.write() `方法会从当前 `ChannelHandler` 开始在 `pipeline` 中向前传播 `write` 事件直到 `HeadContext`。
* `channelHandlerContext.channel().write() `方法则会从 `pipeline` 的尾结点 ``TailContext 开始在 `pipeline` 中向前传播 `write` 事件直到 `HeadContext` 。

注意：这里除了 `write`事件，`flush`事件也是一样的传播方式。

下面以 `channelHandlerContext.write() `方法为例说明。

```java
abstract class AbstractChannelHandlerContext implements ChannelHandlerContext, ResourceLeakHint {
    public ChannelFuture write(Object msg) {
        return write(msg, newPromise());
    }
    public ChannelFuture write(final Object msg, final ChannelPromise promise) {
        write(msg, false, promise);
        return promise;
    }
    private void write(Object msg, boolean flush, ChannelPromise promise) {
        ObjectUtil.checkNotNull(msg, "msg");
        // ...

        // flush=true 表示channelHandler中调用的是writeAndFlush方法，
        // 这里需要找到pipeline中覆盖write或者flush方法的channelHandler
        // flush=false 表示调用的是write方法，只需要找到pipeline中覆盖write方法的channelHandler
        final AbstractChannelHandlerContext next = findContextOutbound(flush ?
                (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
        // 用于检查内存泄漏
        final Object m = pipeline.touch(msg, next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            if (flush) {
                next.invokeWriteAndFlush(m, promise);
            } else {
                next.invokeWrite(m, promise);
            }
        } else {
            final WriteTask task = WriteTask.newInstance(next, m, promise, flush);
            if (!safeExecute(executor, task, promise, m, !flush)) {
                task.cancel();
            }
        }
    }
}
```

当调用write方法时，会返回一个ChannelFuture，当然，我们也可以在这个ChannelFuture中添加ChannelFutureListner，当数据发送到底层socket中时，Netty会通知写入结果。

```java
public void channelRead(final ChannelHandlerContext ctx, final Object msg) {
    //此处的msg就是Netty在read loop中从NioSocketChannel中读取到的ByteBuffer
    ChannelFuture future = ctx.write(msg);
    future.addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            Throwable cause = future.cause();
            if (cause != null) {
                 处理异常情况
            } else {      
                 写入Socket成功后，Netty会通知到这里
            }
        }
    });
}
```

当异步事件在 `pipeline` 传播的过程中发生异常时，异步事件就会停止在 `pipeline` 中传播。所以我们在日常开发中，需要对写操作异常情况进行处理。

* 其中 `inbound` 类异步事件发生异常时， **会触发 `exceptionCaught`事件传播** 。`exceptionCaught `事件本身也是一种 `inbound`事件，传播方向会从当前发生异常的 `ChannelHandler `开始一直向后传播直到 `TailContext`。
* 而 `outbound `类异步事件发生异常时， **则不会触发 `exceptionCaught`事件传播** 。一般只是通知相关 `ChannelFuture`。但如果是 `flush` 事件在传播过程中发生异常，则会触发当前发生异常的 `ChannelHandler` 中 `exceptionCaught `事件回调。

回到主线，首先需要找到一个具有执行资格的 `ChannelHandler`，这里寻找逻辑和之前的事件是类似的，这里不再详述。然后开始向前传播 `write`事件。

首先要判断是否在当前线程，在向 `pipeline` 添加 `ChannelHandler `的时候可以通过 `ChannelPipeline#addLast(EventExecutorGroup,ChannelHandler......)` 方法指定执行该 `ChannelHandler `的 `executor`。如果不特殊指定，那么执行该 `ChannelHandler `的 `executor`默认为该 `Channel `绑定的 `Reactor` 线程。

如果是，那么我们直接在当前线程中执行 `ChannelHandler `中的 `write` 方法。如果不是，我们就需要将 `ChannelHandler` 对 `write` 事件的回调操作封装成异步任务 `WriteTask` 并提交给 `ChannelHandler` 指定的 `executor `中，由 `executor `负责执行。那通过这个逻辑也能明白为什么线程安全了。

这里 `write`事件在传播过程中，前面的所有处理都是如一些编解码等等的处理，而真正将消息 `write`的处理时在 `HeadContext`中。

## 4.1 HeadContext

实际发送数据依赖底层 `AbstractUnsafe`

```java
// AbstractUnsafe
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();
    // 待发送数据缓冲队列  Netty是全异步框架，所以这里需要一个缓冲队列来缓存用户需要发送的数据
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    // ...
    int size;
    try {
        // 过滤message类型,这里只会接受DirectBuffer或者fileRegion类型的msg
        msg = filterOutboundMessage(msg);
        // 计算当前msg大小
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        try {
            ReferenceCountUtil.release(msg);
        } finally {
            safeSetFailure(promise, t);
        }
        return;
    }
    outboundBuffer.addMessage(msg, size, promise);
}
```

## 4.2 ChannelOutboundBuffer

这个缓存对象主要是缓存待发送的数据，封装了待写入 `socket`中的信息，以及 `write`方法中返回给用户的 `ChannelPromise`。另外还有三个重要的指针

```java
// 指向第一个待发送数据的Entry
private Entry unflushedEntry;
// 指向最后一个待发送数据的Entry
private Entry tailEntry;
// 这个其实就是unflushedEntry的一个临时变量，调用flush时会指向unflushedEntry所在的位置
private Entry flushedEntry;
```

### 4.2.1 向channelOutboundBuffer中缓存待发送数据

在调用 `write`方法之后最终会在 `HeadContext`中将待发送的数据写入到 `channel`对应的缓冲区 `ChannelOutboundBuffer`中。

```java
// AbstractUnsafe.java
public final void write(Object msg, ChannelPromise promise) {
    // 待发送数据缓冲队列  Netty是全异步框架，所以这里需要一个缓冲队列来缓存用户需要发送的数据
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    // ...
    int size;
    try {
        // 过滤message类型,这里只会接受DirectBuffer或者fileRegion类型的msg
        msg = filterOutboundMessage(msg);
        // 计算当前msg大小
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        // ...
    }
    outboundBuffer.addMessage(msg, size, promise);
}
// ChannelOutboundBuffer
public void addMessage(Object msg, int size, ChannelPromise promise) {
    // 创建Entry对象来封装待发送数据
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
    }
    tailEntry = entry;
    // 指向 ChannelOutboundBuffer 中第一个未被 flush 进 Socket 
    // 的待发送数据。用来指示 ChannelOutboundBuffer 的第一个节点
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }
    // increment pending bytes after adding message to the unflushed arrays.
    // See https://github.com/netty/netty/issues/1619
    // 更新水位线
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }
    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
    // 超过最高水位线，需要将当前channel设置为不可写，并在pipeline中传播不可写事件
    if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
        setUnwritable(invokeLater);
    }
}
```

这里注意 `Entry`对象会由线程池统一管理，不能随意构造。然后就是传播的不可写事件是 `inbound`事件。我们可以在 `ChannelHandler`中自定义此事件的处理

```java
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {

        if (ctx.channel().isWritable()) {
            ...........当前channel可写.........
        } else {
            ...........当前channel不可写.........
        }
    }
}
```

## 4.3 Entry

`ChannelOutboundBuffer`其实就是一个单向链表，`Entry`就是节点

```java
static final class Entry {
    // Entry对象池，用于创建和回收Entry
    private static final ObjectPool<Entry> RECYCLER = ObjectPool.newPool(new ObjectCreator<Entry>() {
        @Override
        public Entry newObject(Handle<Entry> handle) {
            return new Entry(handle);
        }
    });
    // DefaultHandle用于回收对象
    private final Handle<Entry> handle;
    Entry next;
    // 待发送数据
    Object msg;
    // msg转换为jdk nio 中的byteBuffer
    ByteBuffer[] bufs;
    ByteBuffer buf;
    // 传入的Future
    ChannelPromise promise;
    // 已发送多少数据
    long progress;
    // 总共需要发送的数据，不包含entry对象大小
    long total;
    // pendingSize表示entry对象在堆中需要的内存总量
    // 待发送数据大小 + entry对象本身在堆中占用内存大小（96）
    int pendingSize;
    // msg中包含了几个jdk nio byteBuffer
    int count = -1;
    // 写操作是否被取消
    boolean cancelled;
}
```

### 4.3.1 pendingSize

当由于网络拥塞或者 `Netty `客户端负载很高导致网络数据的接收速度以及处理速度越来越慢，`TCP` 的滑动窗口不断缩小以减少网络数据的发送直到为 0，而 `Netty` 服务端却有大量频繁的写操作，不断的写入到 `ChannelOutboundBuffer` 中。

这样就导致了数据发送不出去但是 `Netty` 服务端又在不停的写数据，慢慢的就会撑爆 `ChannelOutboundBuffer `导致 `OOM`。这里主要指的是堆外内存的 `OOM`，因为 `ChannelOutboundBuffer `中包裹的待发送数据全部存储在堆外内存中。

所以 `Netty` 就必须限制 `ChannelOutboundBuffer` 中的待发送数据的内存占用总量，不能让它无限增长。`Netty` 中定义了高低水位线用来表示 `ChannelOutboundBuffer` 中的待发送数据的内存占用量的上限和下限。注意：这里的内存既包括 `JVM` 堆内存占用也包括堆外内存占用。

* 当待发送数据的内存占用总量超过高水位线的时候，`Netty` 就会将 `NioSocketChannel `的状态标记为不可写状态。否则就可能导致 `OOM`。
* 当待发送数据的内存占用总量低于低水位线的时候，`Netty` 会再次将 `NioSocketChannel `的状态标记为可写状态。

`Netty`使用 `pendingSize`来记录要发送数据占用的总内存。

### 4.3.2 高低水位线

```java
// DefaultChannelConfig
private volatile WriteBufferWaterMark writeBufferWaterMark = WriteBufferWaterMark.DEFAULT;

public final class WriteBufferWaterMark {
    // 低水位线-32KB
    private static final int DEFAULT_LOW_WATER_MARK = 32 * 1024;
    // 高水位线-64KB
    private static final int DEFAULT_HIGH_WATER_MARK = 64 * 1024;
}
```

这也就是说如果数据超过 `64K`时，那 `channel`的状态就是不可写的状态，直到数据低于 `32K`时才可以写。

## 4.4 flush事件

调用 `write`方法之后数据只是被存入到了 `ChannelOutboundBuffer`中，并不会写入到 `Socket`中。所以还需要调用 `flush`方法。`flush`事件的处理和 `write`事件处理基本类似，也是从后往前传播。

触发 `flush` 事件传播的同样也有两个方法：

* `channelHandlerContext.flush()`：`flush`事件会从当前 `channelHandler` 开始在 `pipeline` 中向前传播直到 `headContext`。
* `channelHandlerContext.channel().flush()`：`flush `事件会从 `pipeline` 的尾结点 `tailContext `处开始向前传播直到 `headContext`。

```java
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        //本次OP_READ事件处理完毕
        ctx.flush();
    }
}
```

可以单独调用 `flush`方法，也可以直接调用 `writeAndFlush`方法。

```java
// AbstractChannelHandlerContext
void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
        invokeFlush0();
    } else {
        writeAndFlush(msg, promise);
    }
}
private void invokeFlush0() {
    try {
        ((ChannelOutboundHandler) handler()).flush(this);
    } catch (Throwable t) {
        invokeExceptionCaught(t);
    }
}
```

这里有一点和 `write` 事件处理不同的是，当调用 `nextChannelHandler `的 `flush` 回调出现异常的时候，会触发 `nextChannelHandler` 的 `exceptionCaught` 回调。

```java
private void invokeExceptionCaught(final Throwable cause) {
    if (invokeHandler()) {
        try {
            handler().exceptionCaught(this, cause);
        } catch (Throwable error) {
            if (logger.isDebugEnabled()) {
                logger.debug(....相关日志打印......);
            } else if (logger.isWarnEnabled()) {
                logger.warn(...相关日志打印......));
            }
        }
    } else {
        fireExceptionCaught(cause);
    }
}
```

而其他 `outbound` 类事件比如 `write` 事件在传播的过程中发生异常，只是回调通知相关的 `ChannelFuture`。并不会触发 `exceptionCaught` 事件的传播。

```java
// HeadContext
public void flush(ChannelHandlerContext ctx) {
    unsafe.flush();
}
protected abstract class AbstractUnsafe implements Unsafe {
   @Override
    public final void flush() {
        assertEventLoop();
        ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        //channel以关闭
        if (outboundBuffer == null) {
            return;
        }
        //将flushedEntry指针指向ChannelOutboundBuffer头结点，此时变为即将要flush进Socket的数据队列
        outboundBuffer.addFlush();
        //将待写数据写进Socket
        flush0();
    }
}
```

这里就是将数据从缓存中写入到 `socket`中，当然在调整缓存指针到时候会调整水位线。同时在写入 `socket`之前发送是可以被取消的，通过 `ChannelPromise#setUncancellable`方法，此时会释放相应的堆外内存。

水位线低于最低水位线（`32K`）时，就会将当前 `channel`设置为可写状态，并触发 `ChannelWritabilityChanged` 事件（`inbound`事件）的传播。

```java
protected void flush0() {
    // 是否正在进行flush操作
    if (inFlush0) {
        // Avoid re-entrance
        return;
    }
    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) {
        return;
    }
    inFlush0 = true;
    // channel非激活状态（激活状态：open+connect）
    if (!isActive()) {
        try {
            if (!outboundBuffer.isEmpty()) {
                if (isOpen()) {
                    // channel处理非连接状态，通知promise写入失败，并触发channelWritabilityChanged事件
                    outboundBuffer.failFlushed(new NotYetConnectedException(), true);
                } else {
                    // channel处于关闭状态 通知promise 写入失败 但不触发channelWritabilityChanged事件
                    outboundBuffer.failFlushed(newClosedChannelException(initialCloseCause, "flush0()"), false);
                }
            }
        } finally {
            inFlush0 = false;
        }
        return;
    }
    try {
        // 写入socket
        doWrite(outboundBuffer);
    } catch (Throwable t) {
        handleWriteError(t);
    } finally {
        inFlush0 = false;
    }
}
```

首先看非激活状态下的处理，这里会将 `ChannelOutboundBuffer `中 `flushedEntry `与 `tailEntry `之间的 `Entry` 对象节点全部删除，并释放发送数据占用的内存空间，同时回收 `Entry` 对象实例。下面看真正的数据发送

```java
// NioSocketChannel.java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    SocketChannel ch = javaChannel();
    // 最大写入次数 默认为16
    int writeSpinCount = config().getWriteSpinCount();
    do {
        if (in.isEmpty()) {
            // 没有数据，清理掉OP_WRITE
            clearOpWrite();
            return;
        }
        // SO_SNDBUF设置的发送缓冲区大小 * 2 作为 最大写入字节数
        int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
        // 将ChannelOutboundBuffer中缓存的DirectBuffer转换成JDK NIO 的 ByteBuffer
        // 表示本次循环最多可以从缓存in中转换多少个字节出来，也就是最多能发送多少字节
        // 1024：最多可以转换出1024个nio ByteBuffer
        ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
        // ChannelOutboundBuffer中总共的DirectBuffer数
        int nioBufferCnt = in.nioBufferCount();
        switch (nioBufferCnt) {
            // ...
        }
    } while (writeSpinCount > 0);
    // 处理循环未写完数据的情况
    incompleteWrite(writeSpinCount < 0);
}
```

这里首先是获取配置的循环次数，默认为16次。然后将 `ChannelOutboundBuffer`转换成 `nio buffer`，可能会转换成多个，然后进行实际发送。

```java
switch (nioBufferCnt) {
    case 0:
        // ChannelOutboundBuffer只支持ByteBuf和FileRegion类型数据，这里转换的nio buffer
        // 为0，但是ChannelOutboundBuffer却不为空，那说明有FileRegion类型数据在传输
        // 正在进行网络文件的传输（这里主要针对FileRegion类型数据）
        writeSpinCount -= doWrite0(in);
        break;
    case 1: {
        ByteBuffer buffer = nioBuffers[0];
        int attemptedBytes = buffer.remaining();
        final int localWrittenBytes = ch.write(buffer);
        if (localWrittenBytes <= 0) {
            // 当前socket发送缓存无法写入了，注册OP_WRITE，等待下次处理
            incompleteWrite(true);
            return;
        }
        // 根据当前实际写入情况调整 maxBytesPerGatheringWrite数值
        adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
        // 如果ChannelOutboundBuffer中的某个Entry被全部写入 则删除该Entry
        // 如果Entry被写入了一部分 还有一部分未写入  则更新Entry中的readIndex 等待下次writeLoop继续写入
        in.removeBytes(localWrittenBytes);
        --writeSpinCount;
        break;
    }
    default: {
        long attemptedBytes = in.nioBufferSize();
        // 批量写入
        final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
        if (localWrittenBytes <= 0) {
            incompleteWrite(true);
            return;
        }
        // 根据实际写入情况调整一次写入数据大小的最大值
        // maxBytesPerGatheringWrite决定每次可以从channelOutboundBuffer中获取多少发送数据
        adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                maxBytesPerGatheringWrite);
        // 移除全部写完的BUffer，如果只写了部分数据则更新buffer的readerIndex，下一个writeLoop写入
        in.removeBytes(localWrittenBytes);
        --writeSpinCount;
        break;
    }
}
```

实际发送分为几种情况，其中0是一种专门针对 `FileRegion`数据类型的处理，另外两种情况其实是一样的。

### 4.4.1 FileRegion类型数据处理

`nioBufferCnt`为0，表示没有转换出一个 `nio buffer`，但是 `ChannelOutboundBuffer`又不为空，这说明正在进行网络文件传输，也就是处理 `FileRegion`类型的数据。这里采用零拷贝来进行传输。如果 `socket`缓存无法写入，那么一次循环结束后 `writeSpinCount`会小于0。

### 4.4.2 普通类型数据处理

对于普通类型的数据，则需要转换成 `nio buffer`，可能会转换成一个或者多个，然后进行发送。然后根据每次传输的数据量调整相关参数。而对于已经全部发送出去的 `Entry`，则需要删除。当 `localWrittenBytes`小于等于零时则表示 `socket`缓存无法写入数据了。

回到主线，循环发送数据结束后存在两种情况，一种是 `socket`无法写入了，另外一种是循环达到最大次数限制了，下面要看结尾处理

```java
// AbstractNioByteChannel
protected final void incompleteWrite(boolean setOpWrite) {
    // 如果为true，表示socket无法写入了，可能数据还未写完，此时重新注册OP_WRITE标记
    if (setOpWrite) {
        // socket缓存不可写了，可能还没写16次
        setOpWrite();
    } else {
        // 这里表示socket缓存是可写的，但是如果是已经写了16次
        clearOpWrite();
        // 既然当前 Socket 缓冲区是可写的，我们就不能注册 OP_WRITE 事件，否则这里一直会
        // 不停地收到 epoll 的通知。因为 JDK NIO Selector 默认的是 epoll 的水平触发。
        eventLoop().execute(flushTask);
    }
}
```

这里可以看到，如果 `socket`无法写入了，那么就需要向当前 `NioSocketChannel `对应的 `Reactor `注册 `OP_WRITE` 事件，并停止当前 `flush `流程。当 `Socket `的写缓冲区有容量可写时，`epoll `会通知 `reactor` 线程继续写入。

如果是因为循环次数达到上限，此时就不能直接注册 `OP_WRITE`事件了，`Netty `不会允许 `reactor `线程一直在一个 `channel `上执行 `IO`操作，`reactor` 线程的执行时间需要均匀的分配到每个 `channel` 上。所以这里 `Netty` 会停下，转而去处理其他 `channel `上的 `IO `事件。而同时这里就不能直接注册 `OP_WRITE`事件了，因为次数已经达到上限，如果直接注册，那**这里一直会不停地收到 `epoll `的通知。所以这里只能向 `reactor `提交 `flushTask` 来继续完成剩下数据的写入，而不能注册 `OP_WRITE` 事件。**

但是这里我们会首先清理 `OP_WRITE`事件，是因为当写事件发生后，`Netty`会直接调用 `channel`的 `forceFlush`方法，所以这里需要首先进行清理。

