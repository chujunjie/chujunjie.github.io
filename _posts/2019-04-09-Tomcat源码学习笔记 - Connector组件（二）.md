---
layout:     post
title:      Tomcat源码学习笔记 - Connector组件（二）
subtitle:   
date:       2019-04-30
author:     chujunjie
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - java
    - Tomcat
---



##### 前言

上一篇文章讲到Poller处理完之后，交给SocketProcessor执行处理，这篇就详细记录下这个处理过程。



##### SocketProcessor

SocketProcessor实现Runnable接口，对外暴露run()方法，内部封装doRun()。

```java
protected class SocketProcessor extends SocketProcessorBase<NioChannel> {
    public SocketProcessor(SocketWrapperBase<NioChannel> socketWrapper, SocketEvent event) {
        super(socketWrapper, event);
    }

    @Override
    protected void doRun() {
        NioChannel socket = socketWrapper.getSocket();
        SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());

        try {
            // 这里的 handshake 是用来标记https的握手完成情况，
            // 如果是http不需要该握手阶段，在从c1处直接置为0
            int handshake = -1;
            try {
                if (key != null) {
                    if (socket.isHandshakeComplete()) { // c1
                        handshake = 0;
                    } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                               event == SocketEvent.ERROR) {
                        // 如果不能完成TLS握手过程，标记握手失败
                        handshake = -1;
                    } else {
                        // 处理https的SecureNioChannel覆写了该hanshake()方法
                        // 返回注册的SelectionKey
                        handshake = socket.handshake(key.isReadable(), key.isWritable());
                        event = SocketEvent.OPEN_READ;
                    }
                }
            } catch (IOException x) {
                ...
            }
            if (handshake == 0) { // 握手完成
                SocketState state = SocketState.OPEN; // 标记Socket状态
                // 处理该Sockt里的请求
                if (event == null) {
                    // c2
                    state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                } else {
                    state = getHandler().process(socketWrapper, event);
                }
                if (state == SocketState.CLOSED) {
                    close(socket, key); // 否则关闭通道
                }
            } else if (handshake == -1) { // 握手失败则关闭通道
                close(socket, key);
            } else if (handshake == SelectionKey.OP_READ) { // TLS会走到这里
                socketWrapper.registerReadInterest(); // 注册读就绪
            } else if (handshake == SelectionKey.OP_WRITE) {
                socketWrapper.registerWriteInterest(); // 注册写就绪
            }
        } catch (CancelledKeyException cx) {
            ... // 处理异常
        } finally {
            socketWrapper = null;
            event = null;
            if (running && !paused) {
                processorCache.push(this); // 将SocketProcessor放回缓存中
            }
        }
    }
}
```

在c2处，该方法将包装好的socketWrapper继续传下去，来看下调用栈，最终在Http11Processor的service方法中处理，而这个Http11Processor正是Processor的一个实现类，用来真正解析流。

![1](C:\Users\chujunjie\Desktop\2019-04-09-Tomcat源码学习笔记 - Connector组件（二）\1.png)



##### Http11Processor

先看一下Http11Processor的构造方法。首先调用抽象父类的构造方法，new了一个Request和Response，用于接收解析结果和Servlet处理完的内容，值得注意的是这里的Request和Response是org.apache.coyote包下的，与org.apache.catalina.connector包下的Request和Response不同，拿Request来说，前者是服务器请求的一种底层，但高效的表示，同时它也是GC-free，并且将消耗计算资源的操作作了延迟处理，可以说是按需再取。说它底层且高效是因为它是直接操作数据流，转换成需要的信息，所以它不适合用户代码。而后者是Coyote Request的包装，为用户（也就是servlet）提供了一个高级视图（外观模式）。

并且在c3处可以看到传入了一个Adapter，这个Adapter则是基于coyote的servlet容器中的入口点。也就是说其作用是Connector与Container之间的桥梁。

```java
public Http11Processor(AbstractHttp11Protocol<?> protocol, Adapter adapter) {
    super(adapter);
    this.protocol = protocol; // protocol

    httpParser = new HttpParser(protocol.getRelaxedPathChars(),
            protocol.getRelaxedQueryChars());

    inputBuffer = new Http11InputBuffer(request, protocol.getMaxHttpHeaderSize(),
            protocol.getRejectIllegalHeaderName(), httpParser);
    request.setInputBuffer(inputBuffer); // 输入缓冲，用于解析请求头及传输编码

    outputBuffer = new Http11OutputBuffer(response, protocol.getMaxHttpHeaderSize());
    response.setOutputBuffer(outputBuffer);  // 输出缓冲，写headers和response主题

    inputBuffer.addFilter(new IdentityInputFilter(protocol.getMaxSwallowSize()));
    outputBuffer.addFilter(new IdentityOutputFilter()); // 添加身份认证过滤器
    inputBuffer.addFilter(new ChunkedInputFilter(protocol.getMaxTrailerSize(),
            protocol.getAllowedTrailerHeadersInternal(), protocol.getMaxExtensionSize(),
            protocol.getMaxSwallowSize()));
    outputBuffer.addFilter(new ChunkedOutputFilter()); // 添加分块过滤器
    
    // Void输入输出过滤器，在尝试读取时返回-1。 与GET，HEAD或类似请求一起使用。
    inputBuffer.addFilter(new VoidInputFilter());
    outputBuffer.addFilter(new VoidOutputFilter()); 

    // 输入过滤器负责读取和缓冲请求体，以便它不会干扰客户端SSL握手消息。
    inputBuffer.addFilter(new BufferedInputFilter());

    //inputBuffer.addFilter(new GzipInputFilter());
    outputBuffer.addFilter(new GzipOutputFilter()); // Gzip

    pluggableFilterIndex = inputBuffer.getFilters().length; // 标记过滤器数量
}

public AbstractProcessor(Adapter adapter) {
    this(adapter, new Request(), new Response()); // new了一个Request、Response
}
protected AbstractProcessor(Adapter adapter, Request coyoteRequest, Response coyoteResponse) {
    this.adapter = adapter; // Adapter c3
    asyncStateMachine = new AsyncStateMachine(this);
    request = coyoteRequest;
    response = coyoteResponse;
    response.setHook(this);
    request.setResponse(response);
    request.setHook(this);
    userDataHelper = new UserDataHelper(getLog());
}
```

看完基本的构造方法，大致能了解这个Http11Processor的功能便是解析流，然后封装成Request，最后给Servlet做处理。下面来看下调用栈中提到的service方法。可以看到处理socket分为几个阶段，解析——>准备——>服务——>结束输入、输出——>KEEPALIVE——>结束，当然如果对于keepalive的请求来说，会在前5个阶段循环。前四个阶段表示了一个完成的请求处理及响应。在c4处可以看到获取了Adapter，而这个Adapter的实现类为CoyoteAdapter，这个Adapter将Request和Reponse的处理委托给servlet。

```java
    @Override
    public SocketState service(SocketWrapperBase<?> socketWrapper)
        throws IOException {
       	// 包含有关请求处理的统计、管理信息。
        RequestInfo rp = request.getRequestProcessor();
        rp.setStage(org.apache.coyote.Constants.STAGE_PARSE); // 解析阶段

        setSocketWrapper(socketWrapper); // 初始化I/O设置，
        inputBuffer.init(socketWrapper);
        outputBuffer.init(socketWrapper);

        keepAlive = true; // 标志位
        openSocket = false;
        readComplete = true;
        boolean keptAlive = false;
        SendfileState sendfileState = SendfileState.DONE;
 
        while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null && sendfileState == SendfileState.DONE && !protocol.isPaused()) {
            try {
                // 从buffer中读取并解析请求基本信息，比如方法（GET/POST/PUT等等）、URI、协议等等
                if (!inputBuffer.parseRequestLine(keptAlive, protocol.getConnectionTimeout(), protocol.getKeepAliveTimeout())) {
                    if (inputBuffer.getParsingRequestLinePhase() == -1) {
                        return SocketState.UPGRADING;
                    } else if (handleIncompleteRequestLineRead()) {
                        break;
                    }
                }

                if (protocol.isPaused()) {
                    response.setStatus(503); // 如果协议解析服务被暂停，则返回503
                    setErrorState(ErrorState.CLOSE_CLEAN, null);
                } else {
                    keptAlive = true;
                    // 每次更新header属性的数量限制（可以通过JMX修改）
                    request.getMimeHeaders().setLimit(protocol.getMaxHeaderCount());
                    // 解析Request Headers
                    if (!inputBuffer.parseHeaders()) {
                        openSocket = true;
                        readComplete = false;
                        break;
                    }
                    if (!protocol.getDisableUploadTimeout()) { socketWrapper.setReadTimeout(protocol.getConnectionUploadTimeout());
                    }
                }
            } catch (IOException e) {
                ... // 处理异常
            }
            Enumeration<String> connectionValues = request.getMimeHeaders().values("Connection"); // 从Header里面取到Connection属性
            boolean foundUpgrade = false;
            while (connectionValues.hasMoreElements() && !foundUpgrade) {
                foundUpgrade = connectionValues.nextElement().toLowerCase(
                        Locale.ENGLISH).contains("upgrade");
            }

            if (foundUpgrade) { // 如果包含upgrade，表示协议需要升级
                String requestedProtocol = request.getHeader("Upgrade"); // 获取升级协议
                UpgradeProtocol upgradeProtocol = protocol.getUpgradeProtocol(requestedProtocol); // 包装成UpgradeProtocol
                if (upgradeProtocol != null) {
                    if (upgradeProtocol.accept(request)) {
                        response.setStatus(HttpServletResponse.SC_SWITCHING_PROTOCOLS);
                        response.setHeader("Connection", "Upgrade"); // 设置响应体
                        response.setHeader("Upgrade", requestedProtocol);
                        action(ActionCode.CLOSE,  null);
                        getAdapter().log(request, response, 0);

                        InternalHttpUpgradeHandler upgradeHandler =
                                upgradeProtocol.getInternalUpgradeHandler(
                                        socketWrapper, getAdapter(), cloneRequest(request));
                        UpgradeToken upgradeToken = new UpgradeToken(upgradeHandler, null, null);
                        action(ActionCode.UPGRADE, upgradeToken);
                        return SocketState.UPGRADING;
                    }
                }
            }
            if (getErrorState().isIoAllowed()) {
                // 准备阶段
                rp.setStage(org.apache.coyote.Constants.STAGE_PREPARE);
                try {
                    // 主要设置过滤器，并且解析部分headers
                    // 比如检查是否是keepAlive、如果是http/1.1，判断是否包含Expect：100-continue
                    // (用于客户端在发送POST数据给服务器前，征询服务器情况，
                    // 看服务器是否处理POST的数据，常用于大文件post，例如文件上传)、
                    // 用户代理user-agent状况、host等等
                    prepareRequest();
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    if (log.isDebugEnabled()) {
                        log.debug(sm.getString("http11processor.request.prepare"), t);
                    }
                    // 500 - Internal Server Error
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, t);
                }
            }

            // 如果是长连接，则需要判断长连接是否达到服务器上限
            int maxKeepAliveRequests = protocol.getMaxKeepAliveRequests();
            if (maxKeepAliveRequests == 1) {
                keepAlive = false;
            } else if (maxKeepAliveRequests > 0 &&
                    socketWrapper.decrementKeepAlive() <= 0) {
                keepAlive = false;
            }

            // 通过Adapter交给Container处理请求，并构建Response
            if (getErrorState().isIoAllowed()) {
                try {
                    rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                    getAdapter().service(request, response); // c4
                    if(keepAlive && !getErrorState().isError() && !isAsync() &&
                            statusDropsConnection(response.getStatus())) {
                        setErrorState(ErrorState.CLOSE_CLEAN, null);
                    }
                } catch (InterruptedIOException e) {
                    ... // 处理异常
            }

            // 结束请求处理
            rp.setStage(org.apache.coyote.Constants.STAGE_ENDINPUT);
            if (!isAsync()) {
                // 如果这是异步请求，则请求在完成后结束。
                // 在这种情况下，AsyncContext负责调用endRequest()。
                endRequest();
            }
            rp.setStage(org.apache.coyote.Constants.STAGE_ENDOUTPUT);

            // 确保设置错误码，且更新请求的统计计数
            if (getErrorState().isError()) {
                response.setStatus(500);
            }
            if (!isAsync() || getErrorState().isError()) {
                request.updateCounters();
                if (getErrorState().isIoAllowed()) {
                    inputBuffer.nextRequest();
                    outputBuffer.nextRequest();
                }
            }

            if (!protocol.getDisableUploadTimeout()) {
                int connectionTimeout = protocol.getConnectionTimeout();
                if(connectionTimeout > 0) {
                    socketWrapper.setReadTimeout(connectionTimeout);
                } else {
                    socketWrapper.setReadTimeout(0);
                }
            }

            rp.setStage(org.apache.coyote.Constants.STAGE_KEEPALIVE); // 如果是长连接继续循环

            sendfileState = processSendfile(socketWrapper);
        }

        rp.setStage(org.apache.coyote.Constants.STAGE_ENDED);
        ...
    }
```

Connector的对请求数据流的处理流程基本已经完成，接下去便是交由Container组件去做具体的Servlet处理了。在上一篇博客中提到Tomcat为了高效的处理请求流，做了很多包括数据结构（轻量级的同步栈SynchronizedStack），以及延迟的异常处理等努力。接下来可以看下本篇文章代码中涉及到的一些数据结构等。



##### MessageBytes

我们在上节中提到org.apache.coyote包下的Request是一种底层且高效的表示。因为我们从其final域中可以看到其类型都是MessageBytes类型的。这个类用来表示HTTP消息中的字节数组，可以代表请求/响应中的全部元素。且字节、字符之间的转换都是延迟且有缓存机制，并且是可以回收的。

该对象可以表示byte []，char []或（子）String。也就是说，在接受到socket传入的字节之后并不会马上进行编码转换，而是保持byte[]的方式，在用到的时候再进行转换。

```java
private int type = T_NULL;
public static final int T_NULL = 0; // 表示空消息
public static final int T_STR = 1; // 表示字符串类型
public static final int T_BYTES = 2; // 表示字节数组类型
public static final int T_CHARS = 3; // 表示字符数组类型
```

另外拥有一个ByteChunk类型和一个CharChunk类型的对象，这两个类可以看作是ByteBuffer的扩展，提供对字节/字符数组的操作。

```java
private final ByteChunk byteC = new ByteChunk();
private final CharChunk charC = new CharChunk();
```

在Http11Processor的service方法处理请求数据的时候，经常用到MessageBytes的setBytes()方法，该方法将指定的字节数组存入缓冲区byteC里面。

```java
    public void setBytes(byte[] b, int off, int len) {
        byteC.setBytes(b, off, len);
        type = T_BYTES;
        hasStrValue = false;
        hasHashCode = false;
        hasLongValue = false;
    }
```

既然说MessageBytes提供了一个延迟机制，在没有转换时，请求数据一直以字节形式存储，直到调用toString()才去转换为字符串。

```java
    @Override
    public String toString() {
        if (hasStrValue) {
            return strValue;
        }
        switch (type) {
            case T_CHARS:
                strValue = charC.toString(); // 字符
                hasStrValue = true;
                return strValue;
            case T_BYTES:
                strValue = byteC.toString(); // 字节
                hasStrValue = true;
                return strValue;
        }
        return null;
    }
```

以字节形式为例，调用了ByteChunk的toString()方法。这个方法又调用的StringCache的toString()，这里便用到了缓存机制，这个StringCache的toString()大概就是先判断缓存数组是否存在，如果不存在，则调用toStringInternal()方法，转换好了之后新建缓存并放入数据，如果缓存存在，则先尝试从缓存中取，取不到的话就调用ByteChunk的toStringInternal()方法，从调用栈中也可以看到，代码比较长就不贴了。

```java
    public String toString() {
        if (isNull()) {
            return null;
        } else if (end - start == 0) {
            return "";
        }
        return StringCache.toString(this);
    }
```

![2](C:\Users\chujunjie\Desktop\2019-04-09-Tomcat源码学习笔记 - Connector组件（二）\2.png)

那么这个toStringInternal做了什么事来看一下，正是根据偏移量和待提取长度进行编码提取转换。如果直接使用new String(byte[], int, int, Charset)，将会先copy整个byte数组，这很影响性能，所以先通过decode将数组的一部分编码成CharBuffer，然后在调用new String，非常精髓的一个处理方式，可谓是费尽心思地提高web服务器的性能。

```java
    public String toStringInternal() {
        if (charset == null) {
            charset = DEFAULT_CHARSET;
        }
        CharBuffer cb = charset.decode(ByteBuffer.wrap(buff, start, end - start));
        return new String(cb.array(), cb.arrayOffset(), cb.length());
    }
```

