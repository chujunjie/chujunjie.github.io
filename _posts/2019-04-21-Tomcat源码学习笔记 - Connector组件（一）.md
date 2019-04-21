---
layout:     post
title:      Tomcat源码学习笔记 - Connector组件（一）
subtitle:   
date:       2019-04-21
author:     chujunjie
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - java
    - Tomcat
---



####  前言

Tomcat作为目前非常流行的web容器，其架构设计是非常值得我们借鉴的，它的生命周期管理、多级容器的协调工作，同时在Tomcat中运用了很多设计模式。



##### Connector组件

Tomcat作为一款web容器，响应处理请求，需要与底层数据做交互，而Connector组件就是Service服务与Socket套接字之间的桥梁。Coyote框架是Tomcat默认的Connector，在org.apache.coyote包下，当然我们也可以自己实现自定义的Connector适配。



###### Connector数据结构

关于Connector，有两个非常重要的接口，ProtocolHandler和Adapter，前者用于处理请求，而后者则是Connector与Container容器之间的一个连接器。

protocolHandler是Connector的一个主要属性，是Coyote用来处理协议的主要接口。

```java
public class Connector extends LifecycleMBeanBase  {
	/* 协议Handler */
    protected final ProtocolHandler protocolHandler;
    public Connector() {
        // 默认使用nio
        this("org.apache.coyote.http11.Http11NioProtocol");
    }

    public Connector(String protocol) { // 传入协议名称
        boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
                AprLifecycleListener.getUseAprConnector(); // 是否开启Apr

        if ("HTTP/1.1".equals(protocol) || protocol == null) { // HTTP/1.1
            if (aprConnector) {
                protocolHandlerClassName = "org.apache.coyote.http11.Http11AprProtocol";
            } else {
                protocolHandlerClassName = "org.apache.coyote.http11.Http11NioProtocol";
            }
        } else if ("AJP/1.3".equals(protocol)) { // AJP/1.3
            if (aprConnector) {
                protocolHandlerClassName = "org.apache.coyote.ajp.AjpAprProtocol";
            } else {
                protocolHandlerClassName = "org.apache.coyote.ajp.AjpNioProtocol";
            }
        } else {
            protocolHandlerClassName = protocol; // 其他自定义协议实现
        }
        ProtocolHandler p = null;
        try {
            // 反射获取协议处理类
            Class<?> clazz = Class.forName(protocolHandlerClassName);
            // 实例化
            p = (ProtocolHandler) clazz.getConstructor().newInstance();
        } catch (Exception e) {
            log.error(sm.getString(
                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);
        } finally {
            this.protocolHandler = p;
        }
        ...
    }
}
```

下图是protocolHandler在不同的IO处理及不同协议下的各种实现。

可以看到Coyote默认支持HTTP/1.1、HTTP/2、AJP协议，AJP协议是面向包的协议，因为虽然Tomcat可以作为独立的java web容器，但是它对静态资源的处理速度远不如其他专业的HTTP服务器比如Nginx，所以通常在实际应用中需要与其他HTTP服务器集成，而这个集成由AJP完成，默认监听8009端口。

另外Coyote默认支持NIO、NIO2（AIO）、APR三种I/O处理方式（本文Tomcat版本为9.0.17，Tomcat9删除了BIO的默认支持），APR表示Apache可移植运行库，是Aapche Http服务器的支持库，开启这个模式Tomcat将以JNI的形式调用Apache HTTP服务器的核心链接库来处理文件或网络传输操作，从操作系统级别解决异步IO问题，大幅度的提高服务器的处理和响应性能，是Tomcat运行高并发应用的首选。

![1](https://raw.githubusercontent.com/chujunjie/chujunjie.github.io/master/img/post_img/2019-04-21/1.png)

那么以HTTP/1.1、Tomcat默认使用的nio处理方式为例，先来看看这个Http11NioProtocol，它的唯一构造方法，传入了一个的NioEndpoint对象。

```java
public Http11NioProtocol() {
    super(new NioEndpoint());
}

public AbstractProtocol(AbstractEndpoint<S,?> endpoint) {
    // 每个endpoint对应一种IO策略
    this.endpoint = endpoint;
    // 设置一些默认的属性
    setConnectionLinger(Constants.DEFAULT_CONNECTION_LINGER);
    setTcpNoDelay(Constants.DEFAULT_TCP_NO_DELAY);
}
```



##### NioEndpoint

这个NioEndpoint作为Tomcat NIO的IO处理策略，主要提供工作线程和线程池：

- Socket接收者Acceptor
- Socket轮询者Poller
- 工作线程池



###### NioEndPoint数据结构

从NioEndpoint和其上层的抽象类中可以了解到一些比较重要的属性。

```java
public class NioEndpoint extends AbstractJsseEndpoint<NioChannel, SocketChannel> {
	/* 线程安全的Selector池 */
    private NioSelectorPool selectorPool = new NioSelectorPool();
    /* 轮询事件的缓存栈 */
    private SynchronizedStack<PollerEvent> eventCache;
    /* NioChannel的缓存栈，NioChannel对SocketChannel封装，使SSL与非SSL对外提供相同的处理方式 */
    private SynchronizedStack<NioChannel> nioChannels;
    /* 轮询线程的优先级 */
    private int pollerThreadPriority = Thread.NORM_PRIORITY;
    /* 轮询线程数量，2与JVM可利用线程中取小值 */
    private int pollerThreadCount = 				         Math.min(2,Runtime.getRuntime().availableProcessors());
    /* 轮询线程池 */
	private Poller[] pollers = null;
    ...
}
public abstract class AbstractEndpoint<S,U> {
    /* Endpoint的运行状态 */
    protected volatile boolean running = false;
    /* Endpoint暂停时设置为true */
    protected volatile boolean paused = false;
    /* 标记使用的是否为默认线程池 */
    protected volatile boolean internalExecutor = true;
    /* 流量控制阀门，Socket连接数限制，达到阈值之后放入FIFO队列 */
    private volatile LimitLatch connectionLimitLatch = null;
    /* 代表在Server.xml中可设置的一些Socket相关属性 */
    protected final SocketProperties socketProperties = new SocketProperties();
    /* 接收线程，接收连接并交给工作线程 */
    protected List<Acceptor<U>> acceptors;
    /* Socket处理对象的缓存栈 */
    protected SynchronizedStack<SocketProcessorBase<S>> processorCache;
    /* 接收线程数量，默认为1 */
    protected int acceptorThreadCount = 1;
    /* 接收线程的优先级 */
    protected int acceptorThreadPriority = Thread.NORM_PRIORITY;
    /* 最大连接数nio模式默认10000，Apr模式默认8*1024 */
    private int maxConnections = 10000;
    /* keepAlive时间，没有设置则使用soTimeout */
    private Integer keepAliveTimeout = null;
    /* 是否开启ssl */
    private boolean SSLEnabled = false;
    /* request的keepAlive时间 */
    private int maxKeepAliveRequests = 100;
    /* ConnectionHandler */
    private Handler<S> handler = null;
    /* 用于存放传递给子组件的信息 */
    protected HashMap<String, Object> attributes = new HashMap<>();
}
```

主要包含LimitLatch、Acceptor、Poller、SocketProcessor、Excutor5个部分：

- LimitLatch是一个连接控制器，负责连接限制，nio模式下默认10000，达到阈值则拒绝连接请求。
- Acceptor负责接收请求，默认由1个线程负责，将请求的事件注册到事件列表中。
- Poller负责轮询上述产生的事件，将就绪的事件生成SokcetProcessor，交给Excutor去执行。
- SokcetProcessor里面的doRun方法，封装了Socket的读写，完成Container调用逻辑。
- Excutor用来执行Poller创建的SokcetProcessor，线程池大小Connector节点配置的maxThreads决定。

看到Endpoint的属性中出现了很多SynchronizedStack，这个数据结构为Tomcat量身定做，是ConcurrentLinkedQueue一个GC-free的轻量级替代方案，提供扩容方案，最大128，但没有提供减少容量的方法。减少容量必然带来数组对象的回收，适用于数据量比较固定的场景，另外这个数据结构本身由数组维护，减少了维护节点的开销。

```java
public class SynchronizedStack<T> {

    public static final int DEFAULT_SIZE = 128; // 上限128
    private static final int DEFAULT_LIMIT = -1;
    private int size;
    private final int limit;
    private int index = -1;
    private Object[] stack; // 由一个数组维护

    public SynchronizedStack() {
        this(DEFAULT_SIZE, DEFAULT_LIMIT); // 初始化数组大小
    }
    public SynchronizedStack(int size, int limit) {
        if (limit > -1 && size > limit) {
            this.size = limit;
        } else {
            this.size = size;
        }
        this.limit = limit;
        stack = new Object[size];
    }
    public synchronized boolean push(T obj) { // 放入数据
        index++;
        if (index == size) { // 达到数组最大值，若小于上限值则扩容，否则直接返回放入失败
            if (limit == -1 || size < limit) {
                expand();
            } else {
                index--;
                return false;
            }
        }
        stack[index] = obj;
        return true;
    }

    @SuppressWarnings("unchecked")
    public synchronized T pop() {
        if (index == -1) { // 没有数据返回null
            return null;
        }
        T result = (T) stack[index]; // 返回最后一个数据
        stack[index--] = null;
        return result;
    }

    public synchronized void clear() { // 清空stack
        if (index > -1) {
            for (int i = 0; i < index + 1; i++) {
                stack[i] = null;
            }
        }
        index = -1;
    }

    private void expand() { // 扩容
        int newSize = size * 2;
        if (limit != -1 && newSize > limit) {
            newSize = limit;
        }
        Object[] newStack = new Object[newSize];
        // System.arraycopy是对内存直接进行复制，减少了for循环过程中的寻址时间，从而提高了效能
        System.arraycopy(stack, 0, newStack, 0, size);
        // 这里是唯一产生垃圾的地方，旧数组对象被GC，注意只是数组对象，而不是数组内容对象
        stack = newStack;
        size = newSize;
    }
}
```

而另一个SynchronizedQueue则是一个GC-free的容器，只不过这个是一个FIFO无界容器。



###### NioEndPoint启动

来看下NioEndPoint的startInternal方法，这个方法为Nio Endpoint初始化状态，并创建接收和轮询线程。

```java
public void startInternal() throws Exception {

    if (!running) {
        // 标记运行状态：running
        running = true;
       	// 将暂停标志位掷为false
        paused = false;
        // 创建三个缓存栈，Socket处理缓存栈、轮询事件缓存栈、SocketChannel包装类缓存栈
        processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getProcessorCache());
        eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getEventCache());
        nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getBufferPool());
        // 先获取外部线程池（用户自定义），没有则创建一个内置默认的线程池
        if (getExecutor() == null) {
            createExecutor();
        }
        // 初始化可重入锁LimitLatch，用于限制最大连接数
        initializeConnectionLatch();
        // 根据pollerThreadCount，初始化轮询线程池
        pollers = new Poller[getPollerThreadCount()];
        for (int i = 0; i < pollers.length; i++) {
            pollers[i] = new Poller(); // 创建轮询线程
            Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-" + i);
            pollerThread.setPriority(threadPriority); // 设置线程优先级
            pollerThread.setDaemon(true); // 设置为守护线程
            pollerThread.start(); // 启动线程
        }
		// 创建接收线程
        startAcceptorThreads();
    }
}

protected void startAcceptorThreads() {
    int count = getAcceptorThreadCount(); // 接收线程数量
    acceptors = new ArrayList<>(count);

    for (int i = 0; i < count; i++) {
        Acceptor<U> acceptor = new Acceptor<>(this); 
        String threadName = getName() + "-Acceptor-" + i;
        acceptor.setThreadName(threadName);
        acceptors.add(acceptor);
        // 创建接收线程
        Thread t = new Thread(acceptor, threadName); 
        // 设置线程优先级
        t.setPriority(getAcceptorThreadPriority());
        // 设置接收线程是否为守护，默认为true，可以设为非守护线程
        t.setDaemon(getDaemon());
        // 启动线程
        t.start();
    }
}
```

这个方法比较简单，我们从调用栈中可以看到，这个方法会在Connector的startInternal方法中被调用，而最终在Catilina启动时中被调用。

![2](https://raw.githubusercontent.com/chujunjie/chujunjie.github.io/master/img/post_img/2019-04-21/2.png)



##### Acceptor和Poller

Acceptor和Poller是一个典型的生产者-消费者模式。

###### Acceptor

```java
public class Acceptor<U> implements Runnable {    
    private static final int INITIAL_ERROR_DELAY = 50;
    private static final int MAX_ERROR_DELAY = 1600;
    
    @Override
    public void run() {
        int errorDelay = 0;
        // 循环，直到接收到一个关闭命令
        while (endpoint.isRunning()) {  
            // 循环，如果Endpoint被暂停则循环sleep
            while (endpoint.isPaused() && endpoint.isRunning()) { 
                state = AcceptorState.PAUSED;
                try {
                    Thread.sleep(50); // 50毫秒拉取一次endpoint运行状态
                } catch (InterruptedException e) {
                    // Ignore
                }
            }
            if (!endpoint.isRunning()) {
                break;
            }
            state = AcceptorState.RUNNING;

            try {
                endpoint.countUpOrAwaitConnection(); // 如果达到最大连接数则等待
                if (endpoint.isPaused()) { // 如果等到连接数限制的时候endpoint被暂停，则停止接收
                    continue;
                }
                U socket = null;
                try {
                    socket = endpoint.serverSocketAccept(); // 创建一个socketChannel接收连接
                } catch (Exception ioe) {
                    endpoint.countDownConnection();
                    if (endpoint.isRunning()) {
                        errorDelay = handleExceptionWithDelay(errorDelay); // 延迟异常处理
                        throw ioe; // 重新扔出异常给c1处捕获
                    } else {
                        break;
                    }
                }
                errorDelay = 0; // 成功接收之后重置延时处理异常时间
                if (endpoint.isRunning() && !endpoint.isPaused()) {
                    // setSocketOptions()将Socket传给相应processor处理
                    if (!endpoint.setSocketOptions(socket)) {
                        endpoint.closeSocket(socket);
                    }
                } else {
                    endpoint.destroySocket(socket); // 否则destroy掉该socketChannel
                }
            } catch (Throwable t) { // c1
                ExceptionUtils.handleThrowable(t); // 处理延迟异常
                String msg = sm.getString("endpoint.accept.fail");
                if (t instanceof Error) {
                    ... // 日志记录
                }
            }
        }
        state = AcceptorState.ENDED; // 标记状态为ENDED
    }
       
        protected int handleExceptionWithDelay(int currentErrorDelay) {
        if (currentErrorDelay > 0) {
            try {
                Thread.sleep(currentErrorDelay);
            } catch (InterruptedException e) {
                // Ignore
            }
        }
        // 第一次异常不延迟处理，直接在c2处返回，默认第一次返回50毫秒，后续每次加倍，但最大1.6秒
        if (currentErrorDelay == 0) {
            return INITIAL_ERROR_DELAY; // c2
        } else if (currentErrorDelay < MAX_ERROR_DELAY) {
            return currentErrorDelay * 2;
        } else {
            return MAX_ERROR_DELAY;
        }
    }
}
```

Acceptor在接收请求时非常有意思的一点是采用了一个延时的异常处理机制。在catch一个异常时，第一次不做延迟处理，直接re-throw给下一个catch日志记录，第二次由于currentErrorDelay已经变成50毫秒，该线程睡50毫秒之后继续执行，后续还抛异常则double延迟时间，最大到1.6s，直到正常接收重置该时间。这个机制有效的防止了Acceptor线程不断循环同时抛错，导致cpu资源占用飙升，并且大量重复日志记录，在一些情况下有显著效果，比如：Linux，一个文件的ulimit达到上限，这时线程不断访问该文件，导致连续报错。

而setSocketOptions方法则是处理Acceptor传过来的指定连接。

```java
    protected boolean setSocketOptions(SocketChannel socket) {
        try {
            // 不同于BIO、ARP模式，NIO会采用轮询的方式
            socket.configureBlocking(false);
            Socket sock = socket.socket();
            socketProperties.setProperties(sock); // 设置Socket属性

            NioChannel channel = nioChannels.pop(); // 取出一个NioChannel
            if (channel == null) {
                // 初始化一个BufferHandler的基本属性
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer()); 
                if (isSSLEnabled()) { // 根据是不是SSL包装成不同NioChannel
                    channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
                } else {
                    channel = new NioChannel(socket, bufhandler);
                }
            } else {
                channel.setIOChannel(socket); // 如果缓存中有Niochannel对象可用，则直接复用
                channel.reset();
            }
            getPoller0().register(channel); // 获取可用的循环器，并将nioChannel注册到该Poll上
        } catch (Throwable t) {
            ... // 异常处理
            return false;
        }
        return true;
    }
	// 注册过程
    public void register(final NioChannel socket) {
        socket.setPoller(this);
        // 将NioChannel包装起来
        NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
        socket.setSocketWrapper(ka);
        ka.setPoller(this);
        ka.setReadTimeout(getConnectionTimeout()); // 设置基本属性
        ka.setWriteTimeout(getConnectionTimeout());
        ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
        ka.setSecure(isSSLEnabled());
        PollerEvent r = eventCache.pop(); // 取出一个PollerEvent
        ka.interestOps(SelectionKey.OP_READ);// 注册读就绪事件
        if (r == null) r = new PollerEvent(socket, ka, OP_REGISTER); // 兴趣事件OP_REGISTER
        else r.reset(socket, ka, OP_REGISTER); // PollerEvent不为空则覆盖
        // 调用Poller的addEvent方法，将该事件扔到Poller的
        // SynchronizedQueue<PollerEvent> events这个待处理队列中
        addEvent(r);
    }
    private void addEvent(PollerEvent event) {
        events.offer(event);
        if (wakeupCounter.incrementAndGet() == 0) selector.wakeup(); // 唤醒selector
    }
```

调用selector的wakeup()方法是为了唤醒selector，从Selector的选择方式来说，分为selectNow()、select()、select(timeout)，第一个是非阻塞的，跟wakeup没有关系，但是后两者会等待，select一直等待，select(timeout)设置等待时间。Selector阻塞式选择除了超时、中断之外，只有等到感兴趣事件ready时才会返回，那么这个wakeup()就是构造这样的一个ready场景。

现在，Acceptor已经完成了任务，接下去便是由Poller去处理事件了。

###### Poller

```java
public class Poller implements Runnable {
    /* nio的多路复用选择器 */
    private Selector selector;
    /* 轮询事件的同步队列 */
    private final SynchronizedQueue<PollerEvent> events =
        new SynchronizedQueue<>();
	/* 线程开启状态标记位 */
    private volatile boolean close = false;
    /* 优化过期处理机制 */
    private long nextExpiration = 0;
	/* Selector唤醒计数 */
    private AtomicLong wakeupCounter = new AtomicLong(0);
	/* 就绪通道的数量 */
    private volatile int keyCount = 0;
    
    @Override
    public void run() {
        while (true) { // 循环直到destroy()被调用
            boolean hasEvents = false; // 是否有事件标识
            try {
                if (!close) {
                    // 将events队列，将每个事件中的通道感兴趣的事件注册到Selector中
                    hasEvents = events(); 
                    // 在
                    if (wakeupCounter.getAndSet(-1) > 0) {
					//如果走到了这里，代表已经有就绪的IO通道
                    //调用非阻塞的select方法，直接返回就绪通道的数量
                        keyCount = selector.selectNow();
                    } else {
                        keyCount = selector.select(selectorTimeout);
                    }
                    wakeupCounter.set(0);
                }
                if (close) { // 关闭状态
                    events();
                    timeout(0, false);
                    try {
                        selector.close();
                    } catch (IOException ioe) {
                        log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                    }
                    break;
                }
            } catch (Throwable x) { // 捕获异常
                ExceptionUtils.handleThrowable(x);
                log.error(sm.getString("endpoint.nio.selectorLoopError"), x);
                continue;
            }
            // 如果上面select方法超时，或者被唤醒，则将events队列中的通道注册到Selector上。
            if (keyCount == 0) hasEvents = (hasEvents | events());

            Iterator<SelectionKey> iterator =
                keyCount > 0 ? selector.selectedKeys().iterator() : null;
            // 遍历SelectionKey，并调度活动事件,这个过程跟普通的nio类似
            while (iterator != null && iterator.hasNext()) {
                SelectionKey sk = iterator.next();
                // 获取NioSocketWrapper，如果获取为空，说明其他线程调用了cancelKey()
                // 则去除这个Key，否则调用processKey()
                NioSocketWrapper attachment = (NioSocketWrapper) sk.attachment();
                if (attachment == null) {
                    iterator.remove();
                } else {
                    iterator.remove();
                    processKey(sk, attachment); // 处理SelectKey
                }
            }
            timeout(keyCount, hasEvents);
        }
        getStopLatch().countDown();
    }
}
```

processKey()这个方法主要通过调用processSocket()方法创建一个SocketProcessor，然后丢到Tomcat线程池中去执行。每个Endpoint都有自己的SocketProcessor实现，从Endpoint的属性中可以看到，这个Processor也有缓存机制。