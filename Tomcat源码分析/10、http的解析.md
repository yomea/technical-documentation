分析tomcat对http的解析，我们还是要从tomcat接收到请求开始，我们可以通过浏览器直接请求一下，然后打断点，调试一下。

首先说明，一下分析的是NIO类型的socket处理，大致类图如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/bdd0555ccb7335638961469be137e518.png)

org.apache.tomcat.util.net.NioEndpoint.Acceptor

其run方法

```
public void org.apache.tomcat.util.net.NioEndpoint.Acceptor.run() {

    int errorDelay = 0;

    // Loop until we receive a shutdown command
    while (running) {

       。。。。。。
        try {
            //if we have reached max connections, wait
            countUpOrAwaitConnection();

            SocketChannel socket = null;
            try {
                // Accept the next incoming connection from the server
                // socket
                //阻塞，等待请求
                socket = serverSock.accept();
            } catch (IOException ioe) {
               。。。。。。
            }
            // Successful accept, reset the error delay
            errorDelay = 0;

            // Configure the socket
            if (running && !paused) {
                // setSocketOptions() will hand the socket off to
                // an appropriate processor if successful
                //（*1*）
                if (!setSocketOptions(socket)) {
                    closeSocket(socket);
                }
            } else {
                closeSocket(socket);
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            log.error(sm.getString("endpoint.accept.fail"), t);
        }
    }
    state = AcceptorState.ENDED;
}

 //（*1*）
 protected boolean org.apache.tomcat.util.net.NioEndpoint.setSocketOptions(SocketChannel socket) {
        // Process the connection
        try {
            //disable blocking, APR style, we are gonna be polling it
            //设置为非阻塞
            socket.configureBlocking(false);
            //获取请求的socket
            Socket sock = socket.socket();
            //设置socket参数
            socketProperties.setProperties(sock);
            //从共享对象池中获取一个NioChannel
            NioChannel channel = nioChannels.pop();
            if (channel == null) {
                //创建SocketBufferHandler，用于获取缓冲区，设置可接受的buffer大小，可写的buffer大小
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
                } else {
                    channel = new NioChannel(socket, bufhandler);
                }
            } else {
                channel.setIOChannel(socket);
                channel.reset();
            }
            //将通道包装成事件注册到poller的事件队列中
            //如果有必要的话会立即唤醒selector，然后注册通道到对应的selector中
            getPoller0().register(channel);
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            try {
                log.error("",t);
            } catch (Throwable tt) {
                ExceptionUtils.handleThrowable(tt);
            }
            // Tell to close the socket
            return false;
        }
        return true;
    }
```

新的请求被注册到了poller中，我们来看看poller的run方法

```
public void org.apache.tomcat.util.net.NioEndpoint.Pollerrun() {
    // Loop until destroy() is called
    while (true) {

        boolean hasEvents = false;

        try {
            if (!close) {
                //判断是否有任务
                hasEvents = events();
                //如果添加的任务大于零，那么进行立即获取select，处理掉任务，防止任务堆积
                if (wakeupCounter.getAndSet(-1) > 0) {
                    //if we are here, means we have other stuff to do
                    //do a non blocking select
                    keyCount = selector.selectNow();
                } else {
                    //如果没有啥任务，默认最大select时间为1s
                    keyCount = selector.select(selectorTimeout);
                }
                //设置为零
                wakeupCounter.set(0);
            }
            //如果要求关闭，那么处理调用任务队列中的任务，然后关闭
            if (close) {
                events();
                timeout(0, false);
                try {
                    selector.close();
                } catch (IOException ioe) {
                    log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                }
                break;
            }
        } catch (Throwable x) {
            ExceptionUtils.handleThrowable(x);
            log.error("",x);
            continue;
        }
        //either we timed out or we woke up, process events first
        if ( keyCount == 0 ) hasEvents = (hasEvents | events());

        Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        // Walk through the collection of ready keys and dispatch
        // any active event.
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
            // Attachment may be null if another thread has called
            // cancelledKey()
            if (attachment == null) {
                iterator.remove();
            } else {
                iterator.remove();
                //处理
                processKey(sk, attachment);
            }
        }//while

        //process timeouts
        timeout(keyCount,hasEvents);
    }//while

    getStopLatch().countDown();
}
```

处理key

```
protected void org.apache.tomcat.util.net.NioEndpoint.Poller.processKey(SelectionKey sk, NioSocketWrapper attachment) {
            try {
                if ( close ) {
                    cancelledKey(sk);
                } else if ( sk.isValid() && attachment != null ) {
                    if (sk.isReadable() || sk.isWritable() ) {
                        if ( attachment.getSendfileData() != null ) {
                            processSendfile(sk,attachment, false);
                        } else {
                            unreg(sk, attachment, sk.readyOps());
                            boolean closeSocket = false;
                            // Read goes before write
                            if (sk.isReadable()) {
                                //(*1*)
                                if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                                    closeSocket = true;
                                }
                            }
                           。。。。。。
                           
                           

```
我们还是直接跳到处理http请求协议的方法吧

```
public SocketState org.apache.coyote.http11.Http11Processor.service(SocketWrapperBase<?> socketWrapper)
        throws IOException {
        RequestInfo rp = request.getRequestProcessor();
        rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);

        // Setting up the I/O
        //关联SocketWrapperBase
        setSocketWrapper(socketWrapper);
        //初始化输入缓冲
        inputBuffer.init(socketWrapper);
        //初始化输出缓冲
        outputBuffer.init(socketWrapper);

        // Flags
        keepAlive = true;
        openSocket = false;
        readComplete = true;
        boolean keptAlive = false;
        SendfileState sendfileState = SendfileState.DONE;

        while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null &&
                sendfileState == SendfileState.DONE && !endpoint.isPaused()) {

            // Parsing the request header
            try {
                //解析请求协议
                if (!inputBuffer.parseRequestLine(keptAlive)) {
                    //读取失败时判断是否需要升级
                    if (inputBuffer.getParsingRequestLinePhase() == -1) {
                        return SocketState.UPGRADING;
                        //是否认为请求读取结束
                    } else if (handleIncompleteRequestLineRead()) {
                        break;
                    }
                }
                。。。。。。
            }

            //如果端点已经停止，那么设置服务不可用
            if (endpoint.isPaused()) {
                    // 503 - Service unavailable
                    response.setStatus(503);
                    setErrorState(ErrorState.CLOSE_CLEAN, null);
                } else {
                    keptAlive = true;
                    // Set this every time in case limit has been changed via JMX
                    //设置请求头个数
                    request.getMimeHeaders().setLimit(endpoint.getMaxHeaderCount());
                    //解析请求头
                    if (!inputBuffer.parseHeaders()) {
                        // We've read part of the request, don't recycle it
                        // instead associate it with the socket
                        //如果没有解析完成，也就是数据还没有完全发送过来，那么标识保持这个socket不关闭
                        openSocket = true;
                        //设置读取完成标识为false
                        readComplete = false;
                        //跳出循环，期待下次的通道读取事件
                        break;
                    }
                    。。。。。。
                }
            } 

            。。。。。。

            if (!getErrorState().isError()) {
                // Setting up filters, and parse some request headers
                //记录RequestInfo阶段为准备阶段
                rp.setStage(org.apache.coyote.Constants.STAGE_PREPARE);
                try {
                    //准备请求，内部主要是对解析请求头等信息做进一步的处理
                    prepareRequest();
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    if (log.isDebugEnabled()) {
                        log.debug(sm.getString("http11processor.request.prepare"), t);
                    }
                    // 500 - Internal Server Error
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, t);
                    getAdapter().log(request, response, 0);
                }
            }
            //设置是否需要长连接
            if (maxKeepAliveRequests == 1) {
                keepAlive = false;
            } else if (maxKeepAliveRequests > 0 &&
                    socketWrapper.decrementKeepAlive() <= 0) {
                keepAlive = false;
            }

            // Process the request in the adapter
            if (!getErrorState().isError()) {
                try {
                    //变更请求阶段为服务阶段，表示可以开始提供服务了
                    rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                    //开始调用容器，通过排管一直调用到servletWrapper
                    getAdapter().service(request, response);
                    
                } catch (Throwable t) {
                   
                }
            }

           。。。。。。
    }
```

解析请求协议

```
boolean org.apache.coyote.http11.Http11InputBuffer.parseRequestLine(boolean keptAlive) throws IOException {

        // check state
        if (!parsingRequestLine) {
            return true;
        }
        //
        // Skipping blank lines
        //parsingRequestLinePhase表示解析请求协议的阶段
        if (parsingRequestLinePhase < 2) {
            byte chr = 0;
            do {

                // Read new bytes if needed
                //如果解析的缓冲的开始位置大于等于上限位置，默认tomcat在没有读取任何数据的时候设置的position为零，limit也为零
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (keptAlive) {
                        // Haven't read any request data yet so use the keep-alive
                        // timeout.
                        //设置读取超时时间
                        wrapper.setReadTimeout(wrapper.getEndpoint().getKeepAliveTimeout());
                    }
                    //从通道读取数据
                    if (!fill(false)) {
                        // A read is pending, so no longer in initial state
                        //如果读取到了值，那么设置读取阶段为1
                        parsingRequestLinePhase = 1;
                        return false;
                    }
                    // At least one byte of the request has been received.
                    // Switch to the socket timeout.
                    wrapper.setReadTimeout(wrapper.getEndpoint().getConnectionTimeout());
                }
                if (!keptAlive && byteBuffer.position() == 0 && byteBuffer.limit() >= CLIENT_PREFACE_START.length - 1) {
                    boolean prefaceMatch = true;
                    //是否匹配http2协议头
                    for (int i = 0; i < CLIENT_PREFACE_START.length && prefaceMatch; i++) {
                        if (CLIENT_PREFACE_START[i] != byteBuffer.get(i)) {
                            prefaceMatch = false;
                        }
                    }
                    if (prefaceMatch) {
                        // HTTP/2 preface matched
                        parsingRequestLinePhase = -1;
                        return false;
                    }
                }
                // Set the start time once we start reading data (even if it is
                // just skipping blank lines)
                if (request.getStartTime() < 0) {
                    request.setStartTime(System.currentTimeMillis());
                }
                //获取第一个字符，如果是换行符，那么继续读取，主要是用于跳过空白行
                chr = byteBuffer.get();
            } while ((chr == Constants.CR) || (chr == Constants.LF));
            //因为上面的get方法会使的position加1，这里需要调回去
            byteBuffer.position(byteBuffer.position() - 1);
            //设置解析请求开始位置，刚开始的时候这个位置是零
            parsingRequestLineStart = byteBuffer.position();
            //设置为阶段2
            parsingRequestLinePhase = 2;
            if (log.isDebugEnabled()) {
                log.debug("Received ["
                        + new String(byteBuffer.array(), byteBuffer.position(), byteBuffer.remaining(), StandardCharsets.ISO_8859_1) + "]");
            }
        }
        //阶段2
        if (parsingRequestLinePhase == 2) {
            //
            // Reading the method name
            // Method name is a token
            //
            boolean space = false;
            while (!space) {
                // Read new bytes if needed
                //如果数据已经被处理，或者还未读取，那么尝试从通道中读取数据，以便接下来的处理
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) // request line parsing
                        return false;
                }
                // Spec says method name is a token followed by a single SP but
                // also be tolerant of multiple SP and/or HT.
                int pos = byteBuffer.position();
                byte chr = byteBuffer.get();
                //一直循环读取到的字符，一直寻找到空格或者table符
                if (chr == Constants.SP || chr == Constants.HT) {
                    space = true;
                    //设置消息字节，内部维护字节块，通过偏移地址和结束地址来表示真正需要用的数据
                    //这里截取的是请求方法
                    request.method().setBytes(byteBuffer.array(), parsingRequestLineStart,
                            pos - parsingRequestLineStart);
                //其他情况，判断是否是{或者}这种类型，如果在解析方法时出现这种字符，那么抛出错误
                } else if (!HttpParser.isToken(chr)) {
                    byteBuffer.position(byteBuffer.position() - 1);
                    throw new IllegalArgumentException(sm.getString("iib.invalidmethod"));
                }
            }
            //设置解析阶段为3，表示解析方法阶段的结束
            parsingRequestLinePhase = 3;
        }
        //第三阶段，此阶段主要是跳过空格和table
        if (parsingRequestLinePhase == 3) {
            // Spec says single SP but also be tolerant of multiple SP and/or HT
            boolean space = true;
            while (space) {
                // Read new bytes if needed
                //如果没有数据，那么尝试从通道中读取数据，确保有数据可以继续处理，返回直接返回false，等待下次通道的read事件
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) // request line parsing
                        return false;
                }
                byte chr = byteBuffer.get();
                //直到不是空格或者table
                if (!(chr == Constants.SP || chr == Constants.HT)) {
                    space = false;
                    byteBuffer.position(byteBuffer.position() - 1);
                }
            }
            parsingRequestLineStart = byteBuffer.position();
            parsingRequestLinePhase = 4;
        }
        //第四阶段：解析uri和queryString
        if (parsingRequestLinePhase == 4) {
            // Mark the current buffer position

            int end = 0;
            //
            // Reading the URI
            //
            boolean space = false;
            while (!space) {
                // Read new bytes if needed
                ///如果没有数据，那么尝试从通道中读取数据，确保有数据可以继续处理，返回直接返回false，等待下次通道的read事件
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) // request line parsing
                        return false;
                }
                int pos = byteBuffer.position();
                byte chr = byteBuffer.get();
                //循环读取到空白符合table符号，如果读取到了这些符号，那么本阶段结束，也就是查询字符串已经读取完毕
                if (chr == Constants.SP || chr == Constants.HT) {
                    space = true;
                    end = pos;
                    //换行符，表示到头了
                } else if (chr == Constants.CR || chr == Constants.LF) {
                    // HTTP/0.9 style request
                    parsingRequestLineEol = true;
                    space = true;
                    end = pos;
                    //如果读取到了问号，恭喜，可以解析uri和查询参数了
                } else if (chr == Constants.QUESTION && parsingRequestLineQPos == -1) {
                    parsingRequestLineQPos = pos;
                    //如果读取到了问号并且确认它就是问号，如果不是，那么抛错
                } else if (parsingRequestLineQPos != -1 && !httpParser.isQueryRelaxed(chr)) {
                    // %nn decoding will be checked at the point of decoding
                    throw new IllegalArgumentException(sm.getString("iib.invalidRequestTarget"));
                } else if (httpParser.isNotRequestTargetRelaxed(chr)) {
                    // This is a general check that aims to catch problems early
                    // Detailed checking of each part of the request target will
                    // happen in Http11Processor#prepareRequest()
                    throw new IllegalArgumentException(sm.getString("iib.invalidRequestTarget"));
                }
            }
            //是否读取到了问号
            if (parsingRequestLineQPos >= 0) {
                //圈定查询参数范围
                request.queryString().setBytes(byteBuffer.array(), parsingRequestLineQPos + 1,
                        end - parsingRequestLineQPos - 1);
                //圈定uri范围
                request.requestURI().setBytes(byteBuffer.array(), parsingRequestLineStart,
                        parsingRequestLineQPos - parsingRequestLineStart);
            } else {
                //圈定uri范围
                request.requestURI().setBytes(byteBuffer.array(), parsingRequestLineStart,
                        end - parsingRequestLineStart);
            }
            parsingRequestLinePhase = 5;
        }
        //第五阶段：跳过空白符
        if (parsingRequestLinePhase == 5) {
            // Spec says single SP but also be tolerant of multiple and/or HT
            boolean space = true;
            while (space) {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) // request line parsing
                        return false;
                }
                byte chr = byteBuffer.get();
                if (!(chr == Constants.SP || chr == Constants.HT)) {
                    space = false;
                    byteBuffer.position(byteBuffer.position() - 1);
                }
            }
            parsingRequestLineStart = byteBuffer.position();
            parsingRequestLinePhase = 6;

            // Mark the current buffer position
            end = 0;
        }
        //第6阶段：解析协议
        if (parsingRequestLinePhase == 6) {
            //
            // Reading the protocol
            // Protocol is always "HTTP/" DIGIT "." DIGIT
            //一直读取到行的结尾
            while (!parsingRequestLineEol) {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) // request line parsing
                        return false;
                }

                int pos = byteBuffer.position();
                byte chr = byteBuffer.get();
                //回车符
                if (chr == Constants.CR) {
                    end = pos;
                    //换行符
                } else if (chr == Constants.LF) {
                    if (end == 0) {
                        end = pos;
                    }
                    //标记读取的行已经结束
                    parsingRequestLineEol = true;
                    //是否是协议字符，比如http/1.1
                } else if (!HttpParser.isHttpProtocol(chr)) {
                    throw new IllegalArgumentException(sm.getString("iib.invalidHttpProtocol"));
                }
            }
            //圈定协议字符串范围
            if ((end - parsingRequestLineStart) > 0) {
                request.protocol().setBytes(byteBuffer.array(), parsingRequestLineStart,
                        end - parsingRequestLineStart);
            } else {
                //如果没有协议，设置为空字符串
                request.protocol().setString("");
            }
            //重置解析的行尾false，用于下次解析
            parsingRequestLine = false;
            //解析阶段重置为0
            parsingRequestLinePhase = 0;
            //行结束标识重置为false
            parsingRequestLineEol = false;
            //行读取的字符小标位置重置为零
            parsingRequestLineStart = 0;
            return true;
        }
        throw new IllegalStateException(
                "Invalid request line parse phase:" + parsingRequestLinePhase);
    }
```
上面的代码中有这么一个类HttpParser，下面是它的静态块

```
 static {
        //ARRAY_SIZE = 128
        for (int i = 0; i < ARRAY_SIZE; i++) {
            // Control> 0-31, 127
            if (i < 32 || i == 127) {
                IS_CONTROL[i] = true;
            }

            // Separator
            if (    i == '(' || i == ')' || i == '<' || i == '>'  || i == '@'  ||
                    i == ',' || i == ';' || i == ':' || i == '\\' || i == '\"' ||
                    i == '/' || i == '[' || i == ']' || i == '?'  || i == '='  ||
                    i == '{' || i == '}' || i == ' ' || i == '\t') {
                IS_SEPARATOR[i] = true;
            }

            // Token: Anything 0-127 that is not a control and not a separator
            if (!IS_CONTROL[i] && !IS_SEPARATOR[i] && i < 128) {
                IS_TOKEN[i] = true;
            }

            // Hex: 0-9, a-f, A-F
            if ((i >= '0' && i <='9') || (i >= 'a' && i <= 'f') || (i >= 'A' && i <= 'F')) {
                IS_HEX[i] = true;
            }

            // Not valid for HTTP protocol
            // "HTTP/" DIGIT "." DIGIT
            if (i == 'H' || i == 'T' || i == 'P' || i == '/' || i == '.' || (i >= '0' && i <= '9')) {
                IS_HTTP_PROTOCOL[i] = true;
            }

            if (i >= '0' && i <= '9') {
                IS_NUMERIC[i] = true;
            }

            if (i >= 'a' && i <= 'z' || i >= 'A' && i <= 'Z') {
                IS_ALPHA[i] = true;
            }

            if (IS_ALPHA[i] || IS_NUMERIC[i] || i == '-' || i == '.' || i == '_' || i == '~') {
                IS_UNRESERVED[i] = true;
            }

            if (i == '!' || i == '$' || i == '&' || i == '\'' || i == '(' || i == ')' || i == '*' ||
                    i == '+' || i == ',' || i == ';' || i == '=') {
                IS_SUBDELIM[i] = true;
            }

            // userinfo    = *( unreserved / pct-encoded / sub-delims / ":" )
            if (IS_UNRESERVED[i] || i == '%' || IS_SUBDELIM[i] || i == ':') {
                IS_USERINFO[i] = true;
            }

            // The characters that are normally not permitted for which the
            // restrictions may be relaxed when used in the path and/or query
            // string
            if (i == '\"' || i == '<' || i == '>' || i == '[' || i == '\\' || i == ']' ||
                    i == '^' || i == '`'  || i == '{' || i == '|' || i == '}') {
                IS_RELAXABLE[i] = true;
            }
        }

        String prop = System.getProperty("tomcat.util.http.parser.HttpParser.requestTargetAllow");
        if (prop != null) {
            for (int i = 0; i < prop.length(); i++) {
                char c = prop.charAt(i);
                if (c == '{' || c == '}' || c == '|') {
                    REQUEST_TARGET_ALLOW[c] = true;
                } else {
                    log.warn(sm.getString("http.invalidRequestTargetCharacter",
                            Character.valueOf(c)));
                }
            }
        }

        DEFAULT = new HttpParser(null, null);
    }
```
对应字符的位置的数组值为true，如果我们要判断某些字符是否表示http协议，那么可以通过对应的IS_HTTP_PROTOCOL数组类判断，如果连续都是它，那么就是http协议了，总结一下上面一段代码所做的事情，假设有以下请求
```
GET /index.html?name=xxx&password=123456 HTTP/1.1
Host: www.xxxx.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6

请求体数据
```
当触发read事件时，tomcat需要对这个请求的http协议进行解析，首先先解析协议，分以下阶段解析：
- 第一阶段：循环跳过空白符
- 第二阶段：解析方法，比如这里的GET方法
- 第三阶段：跳过空白符
- 第四阶段：解析uri和查询字段，这里的uri就是/index.html，查询字段就是name=xxx&password=123456
- 第五阶段：跳过空白符
- 第六阶段：解析协议，这里就是HTTP/1.1

如果在读取的过程中，数据不够，无法完整的读取完开始行，那么将socket状态设置为SocketState.LONG，tomcat重新注册读事件，等待下次触发read事件


好了，接下来继续看到org.apache.coyote.http11.Http11Processor的service(SocketWrapperBase<?>)方法中解析请求的方法

```
boolean org.apache.coyote.http11.Http11InputBuffer.parseHeaders() throws IOException {
    if (!parsingHeader) {
        throw new IllegalStateException(sm.getString("iib.parseheaders.ise.error"));
    }
    //设置请求头解析状态为还有更多的请求头需要解析
    HeaderParseStatus status = HeaderParseStatus.HAVE_MORE_HEADERS;

    do {
        //解析请求头
        status = parseHeader();
        //如果缓冲区读取的请求头已经超过最大限制，抛错，如果剩余容量不够存下socketReadBufferSize个字节，那么也会抛错错误
        if (byteBuffer.position() > headerBufferSize || byteBuffer.capacity() - byteBuffer.position() < socketReadBufferSize) {
            throw new IllegalArgumentException(sm.getString("iib.requestheadertoolarge.error"));
        }
    } while (status == HeaderParseStatus.HAVE_MORE_HEADERS);
    //表示读取完成
    if (status == HeaderParseStatus.DONE) {
        parsingHeader = false;
        //将最后的写入位置position，设置为结束位置
        end = byteBuffer.position();
        return true;
    } else {
        return false;
    }
}

```

解析请求头逻辑

```
private HeaderParseStatus org.apache.coyote.http11.Http11InputBuffer.parseHeader() throws IOException {

    //
    // Check for blank line
    //

    byte chr = 0;
    //请求头解析位置，在构建Http11InputBuffer的构造器中，赋了初始值为HeaderParsePosition.HEADER_START
    while (headerParsePos == HeaderParsePosition.HEADER_START) {

        // Read new bytes if needed
        if (byteBuffer.position() >= byteBuffer.limit()) {
            //尝试从通道中读取数据
            if (!fill(false)) {// parse header
                headerParsePos = HeaderParsePosition.HEADER_START;
                //没有读取到数据时，设置解析状态为需要更多的数据
                return HeaderParseStatus.NEED_MORE_DATA;
            }
        }
        //获取字符
        chr = byteBuffer.get();
        //如果是回车符，跳过，期待下一个换行符
        if (chr == Constants.CR) {
            // Skip
            //如果读取到了换行符，表示请求头解析结束，根据http协议的规范，请求结束后会有一个空行表示结束
        } else if (chr == Constants.LF) {
            return HeaderParseStatus.DONE;
            //其他字符
        } else {
            //复位被get吞掉的字符
            byteBuffer.position(byteBuffer.position() - 1);
            //跳出循环
            break;
        }

    }
    //
    if (headerParsePos == HeaderParsePosition.HEADER_START) {
        // Mark the current buffer position
        //标记当前解析的位置
        headerData.start = byteBuffer.position();
        //设置解析点为处理请求头名字
        headerParsePos = HeaderParsePosition.HEADER_NAME;
    }

    //
    // Reading the header name
    // Header name is always US-ASCII
    //
    //解析请求头name
    while (headerParsePos == HeaderParsePosition.HEADER_NAME) {

        // Read new bytes if needed
        if (byteBuffer.position() >= byteBuffer.limit()) {
            if (!fill(false)) { // parse header
                return HeaderParseStatus.NEED_MORE_DATA;
            }
        }

        int pos = byteBuffer.position();
        chr = byteBuffer.get();
        //如果是冒号
        if (chr == Constants.COLON) {
            //那么读取请求头值的开始
            headerParsePos = HeaderParsePosition.HEADER_VALUE_START;
            //记录请求头名字，然后返回与其相对的MimeHeaderField的value（buffer）对象
            headerData.headerValue = headers.addValue(byteBuffer.array(), headerData.start,
                    pos - headerData.start);
            pos = byteBuffer.position();
            // Mark the current buffer position
            //记录当前冒号的位置
            headerData.start = pos;
            headerData.realPos = pos;
            headerData.lastSignificantChar = pos;
            break;
            //如果不是普通字符，a-zA-Z，像数字啊，（，？，：,@等字符就不属于token字符
            //如果不是token字符，那么跳过这行
        } else if (!HttpParser.isToken(chr)) {
            // Non-token characters are illegal in header names
            // Parsing continues so the error can be reported in context
            headerData.lastSignificantChar = pos;
            byteBuffer.position(byteBuffer.position() - 1);
            // skipLine() will handle the error
            return skipLine();
        }

        // chr is next byte of header name. Convert to lowercase.
        //将大写的字符，设置为小写的字符
        if ((chr >= Constants.A) && (chr <= Constants.Z)) {
            byteBuffer.put(pos, (byte) (chr - Constants.LC_OFFSET));
        }
    }

    // Skip the line and ignore the header
    //如果是跳过状态，那么跳过行，为什么在这里又要判断一遍，因为从通道读取的字符不一定是完整的
    //下次通道触发读取事件的时候可以接着这个状态执行
    if (headerParsePos == HeaderParsePosition.HEADER_SKIPLINE) {
        return skipLine();
    }

    //
    // Reading the header value (which can be spanned over multiple lines)
    //
    //开始处理值
    while (headerParsePos == HeaderParsePosition.HEADER_VALUE_START ||
           headerParsePos == HeaderParsePosition.HEADER_VALUE ||
           headerParsePos == HeaderParsePosition.HEADER_MULTI_LINE) {

        if (headerParsePos == HeaderParsePosition.HEADER_VALUE_START) {
            // Skipping spaces
            //跳过空白符
            while (true) {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) {// parse header
                        // HEADER_VALUE_START
                        return HeaderParseStatus.NEED_MORE_DATA;
                    }
                }

                chr = byteBuffer.get();
                //如果不是空格和table符，break，并设置解析点为处理值
                if (!(chr == Constants.SP || chr == Constants.HT)) {
                    headerParsePos = HeaderParsePosition.HEADER_VALUE;
                    byteBuffer.position(byteBuffer.position() - 1);
                    break;
                }
            }
        }
        if (headerParsePos == HeaderParsePosition.HEADER_VALUE) {
            
            // Reading bytes until the end of the line
            //标识是否读到行的末尾
            boolean eol = false;
            while (!eol) {

                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) {// parse header
                        // HEADER_VALUE
                        return HeaderParseStatus.NEED_MORE_DATA;
                    }
                }

                chr = byteBuffer.get();
                //回车
                if (chr == Constants.CR) {
                    // Skip
                //换行符
                } else if (chr == Constants.LF) {
                    eol = true;
                    //如果是空白字符，有人说了，上面不是已经跳过空白符了吗？
                    //注意，上面跳过的是：与值之间的空白符，未跳过值后面的空白符，比如Host:    www.b    aidu.  com    \r\n
                    //它只是跳过了Host:与www之间的空白符
                } else if (chr == Constants.SP || chr == Constants.HT) {
                    //重新填充数据
                    byteBuffer.put(headerData.realPos, chr);
                    //实际位置（何为实际位置，就是去除空白后的位置，空白不认为是实际的位置）
                    headerData.realPos++;
                } else {
                    //覆盖空白符，填充实际位置
                    byteBuffer.put(headerData.realPos, chr);
                    headerData.realPos++;
                    //记录上次能够读取到非空白字符的位置
                    //值得注意，像上面的www.b    aidu.  com，对于realPos会记录到\r\n的前面空白符位置
                    //但是lastSignificantChar记录的却是m的真实位置
                    headerData.lastSignificantChar = headerData.realPos;
                }
            }

            // Ignore whitespaces at the end of the line
            //这地方挺有意思的，上面刚分析完www.b    aidu.  com    \r\n这种字符，它的realPos会指定到\r的前面的空白符字符的位置
            //这里就是直接咔嚓变成了com的这个m的位置，但是记住byteBuffer中依然储存了com后面的空白字符
            headerData.realPos = headerData.lastSignificantChar;

            // Checking the first character of the new line. If the character
            // is a LWS, then it's a multiline header
            //新的一行
            headerParsePos = HeaderParsePosition.HEADER_MULTI_LINE;
        }
        // Read new bytes if needed
        //如果没有值可读了，那么尝试从通道中读取值，如果没有读取到值，那么返回，等待下次通道的读事件发生
        if (byteBuffer.position() >= byteBuffer.limit()) {
            if (!fill(false)) {// parse header
                // HEADER_MULTI_LINE
                return HeaderParseStatus.NEED_MORE_DATA;
            }
        }

        chr = byteBuffer.get(byteBuffer.position());
        if (headerParsePos == HeaderParsePosition.HEADER_MULTI_LINE) {
            //如果不是空白符，那么可以肯定我们这次读取的字符超过了一行，那么可以从头开始继续解析下一行
            if ((chr != Constants.SP) && (chr != Constants.HT)) {
                headerParsePos = HeaderParsePosition.HEADER_START;
                break;
            } else {
                // Copying one extra space in the buffer (since there must
                // be at least one space inserted between the lines)
                //其他情况，继续记录空白符
                byteBuffer.put(headerData.realPos, chr);
                headerData.realPos++;
                headerParsePos = HeaderParsePosition.HEADER_VALUE_START;
            }
        }
    }
    // Set the header value
    //设置请求头值，headerData.start记录的为当前冒号所在的位置，长度为上次获取到的非空白字符位置减去冒号的位置
    headerData.headerValue.setBytes(byteBuffer.array(), headerData.start,
            headerData.lastSignificantChar - headerData.start);
    //复位（本次读取结束，需要复位数据，以便下次读取的开始）
    headerData.recycle();
    return HeaderParseStatus.HAVE_MORE_HEADERS;
}
```
解析请求头，分为这么几个解析状态：
- HeaderParseStatus.DONE：表示解析完成
- HeaderParseStatus.HAVE_MORE_HEADERS：表示还有请求头需要继续解析
- HeaderParseStatus.NEED_MORE_DATA：表示数据不够，需要继续监听read事件获取数据

解析位置有这么几个状态：
- HeaderParsePosition.HEADER_START：表示某行请求头的开始，如果是请求头结束的那一行，那么这个阶段就会读取到回车换行符，解析状态修改为HeaderParseStatus.DONE
- HeaderParsePosition.HEADER_NAME：这个阶段表示读取请求头的名字，比如请求头名字host
- HeaderParsePosition.HEADER_VALUE_START：这个阶段是读取完请求头名之后的一个阶段，这个阶段主要用于跳过冒号和请求头值之间的空白符
- HeaderParsePosition.HEADER_VALUE：这个阶段开始读取请求头值，会去除右边的空白符
- HeaderParsePosition.HEADER_MULTI_LINE：这个阶段表示我们已经解析完一行请求头了，可以检查是否继续读下一行，如果数据够的话，break，重新进入HEADER_START阶段
- HeaderParsePosition.HEADER_SKIPLINE：跳过某行，一般出现非法字符的时候，这行会被跳过

读取完请求头后，设置请求头的结束位置，这个位置又是请求体的开始位置

当tomcat解析完http协议后，需要对解析好的数据做进一步的处理

```
private void org.apache.coyote.http11.Http11Processor.prepareRequest() {

    http11 = true;
    http09 = false;
    contentDelimitation = false;
    //是否支持ssl请求
    if (endpoint.isSSLEnabled()) {
        //设置请求协议为https
        request.scheme().setString("https");
    }
    //获取协议数据块
    MessageBytes protocolMB = request.protocol();
    //判断是否为http1.1
    if (protocolMB.equals(Constants.HTTP_11)) {
        http11 = true;
        protocolMB.setString(Constants.HTTP_11);
    } else if (protocolMB.equals(Constants.HTTP_10)) {
        http11 = false;
        keepAlive = false;
        protocolMB.setString(Constants.HTTP_10);
    } else if (protocolMB.equals("")) {
        // HTTP/0.9
        http09 = true;
        http11 = false;
        keepAlive = false;
    } else {
        // Unsupported protocol
        http11 = false;
        // Send 505; Unsupported HTTP version
        //设置错误码，不支持的http版本
        response.setStatus(505);
        setErrorState(ErrorState.CLOSE_CLEAN, null);
        if (log.isDebugEnabled()) {
            log.debug(sm.getString("http11processor.request.prepare")+
                      " Unsupported HTTP version \""+protocolMB+"\"");
        }
    }
    //获取请求头
    MimeHeaders headers = request.getMimeHeaders();

    // Check connection header
    //获取connect请求头属性
    MessageBytes connectionValueMB = headers.getValue(Constants.CONNECTION);
    if (connectionValueMB != null) {
        ByteChunk connectionValueBC = connectionValueMB.getByteChunk();
        //connect的值中是否包含close字符串，如果包含，那么长连接设置为false
        if (findBytes(connectionValueBC, Constants.CLOSE_BYTES) != -1) {
            keepAlive = false;
            //是否包含keep-alive字符串
        } else if (findBytes(connectionValueBC,
                             Constants.KEEPALIVE_BYTES) != -1) {
            //设置长连接
            keepAlive = true;
        }
    }
    
    if (http11) {
        //对这个属性不了解，跳过
        MessageBytes expectMB = headers.getValue("expect");
        if (expectMB != null) {
            if (expectMB.indexOfIgnoreCase("100-continue", 0) != -1) {
                inputBuffer.setSwallowInput(false);
                request.setExpectation(true);
            } else {
                response.setStatus(HttpServletResponse.SC_EXPECTATION_FAILED);
                setErrorState(ErrorState.CLOSE_CLEAN, null);
            }
        }
    }

    // Check user-agent header
    //处理user-agent属性
    if (restrictedUserAgents != null && (http11 || keepAlive)) {
        MessageBytes userAgentValueMB = headers.getValue("user-agent");
        // Check in the restricted list, and adjust the http11
        // and keepAlive flags accordingly
        if(userAgentValueMB != null) {
            String userAgentValue = userAgentValueMB.toString();
            if (restrictedUserAgents != null &&
                    restrictedUserAgents.matcher(userAgentValue).matches()) {
                http11 = false;
                keepAlive = false;
            }
        }
    }


    // Check host header
    MessageBytes hostValueMB = null;
    try {
        //获取host属性
        hostValueMB = headers.getUniqueValue("host");
    } catch (IllegalArgumentException iae) {
        // Multiple Host headers are not permitted
        // 400 - Bad request
        response.setStatus(400);
        setErrorState(ErrorState.CLOSE_CLEAN, null);
        if (log.isDebugEnabled()) {
            log.debug(sm.getString("http11processor.request.multipleHosts"));
        }
    }
    //如果没有指定从何处来的请求，抛出400，错误的请求
    if (http11 && hostValueMB == null) {
        // 400 - Bad request
        response.setStatus(400);
        setErrorState(ErrorState.CLOSE_CLEAN, null);
        if (log.isDebugEnabled()) {
            log.debug(sm.getString("http11processor.request.noHostHeader"));
        }
    }

    // Check for an absolute-URI less the query string which has already
    // been removed during the parsing of the request line
    //处理uri
    ByteChunk uriBC = request.requestURI().getByteChunk();
    byte[] uriB = uriBC.getBytes();
    //http开头
    if (uriBC.startsWithIgnoreCase("http", 0)) {
        int pos = 4;
        // Check for https
        //https开头
        if (uriBC.startsWithIgnoreCase("s", pos)) {
            pos++;
        }
        // Next 3 characters must be "://"
        //http(s)://开头
        if (uriBC.startsWith("://", pos)) {
            pos += 3;
            //uri在数据块中的偏移位置
            int uriBCStart = uriBC.getStart();

            // '/' does not appear in the authority so use the first
            // instance to split the authority and the path segments
            //寻找到http(s)://后面字符的第一个/
            int slashPos = uriBC.indexOf('/', pos);
            // '@' in the authority delimits the userinfo
            //获取at的位置，一般邮箱地址会这个
            int atPos = uriBC.indexOf('@', pos);
            if (slashPos > -1 && atPos > slashPos) {
                // First '@' is in the path segments so no userinfo
                atPos = -1;
            }
            如果没有/,比如请求地址为https://www.baidu.com，那么设置uri为/
            if (slashPos == -1) {
                slashPos = uriBC.getLength();
                // Set URI as "/". Use 6 as it will always be a '/'.
                // 01234567
                // http://
                // https://
                request.requestURI().setBytes(uriB, uriBCStart + 6, 1);
            } else {
                //其他情况，比如：https://www.baidu.com/index,那么uri为/index
                request.requestURI().setBytes(uriB, uriBCStart + slashPos, uriBC.getLength() - slashPos);
            }

            // Skip any user info
            //跳过用户信息
            if (atPos != -1) {
                // Validate the userinfo
                for (; pos < atPos; pos++) {
                    byte c = uriB[uriBCStart + pos];
                    if (!HttpParser.isUserInfo(c)) {
                        // Strictly there needs to be a check for valid %nn
                        // encoding here but skip it since it will never be
                        // decoded because the userinfo is ignored
                        response.setStatus(400);
                        setErrorState(ErrorState.CLOSE_CLEAN, null);
                        if (log.isDebugEnabled()) {
                            log.debug(sm.getString("http11processor.request.invalidUserInfo"));
                        }
                        break;
                    }
                }
                // Skip the '@'
                pos = atPos + 1;
            }

            if (http11) {
                // Missing host header is illegal but handled above
                if (hostValueMB != null) {
                    // Any host in the request line must be consistent with
                    // the Host header
                    //请求头中的host属性与url中解析的host不同
                    if (!hostValueMB.getByteChunk().equals(
                            uriB, uriBCStart + pos, slashPos - pos)) {
                            //是否允许不匹配的情况
                        if (allowHostHeaderMismatch) {
                            // The requirements of RFC 2616 are being
                            // applied. If the host header and the request
                            // line do not agree, the request line takes
                            // precedence
                            hostValueMB = headers.setValue("host");
                            //以url中的host为准
                            hostValueMB.setBytes(uriB, uriBCStart + pos, slashPos - pos);
                        } else {
                            // The requirements of RFC 7230 are being
                            // applied. If the host header and the request
                            // line do not agree, trigger a 400 response.
                            //不允许的话，只能报错了
                            response.setStatus(400);
                            setErrorState(ErrorState.CLOSE_CLEAN, null);
                            if (log.isDebugEnabled()) {
                                log.debug(sm.getString("http11processor.request.inconsistentHosts"));
                            }
                        }
                    }
                }
            } else {
                // Not HTTP/1.1 - no Host header so generate one since
                // Tomcat internals assume it is set
                //不是http1.1协议的情况，host的值从url中获取
                hostValueMB = headers.setValue("host");
                hostValueMB.setBytes(uriB, uriBCStart + pos, slashPos - pos);
            }
        } else {
            //设置错误编码
            response.setStatus(400);
            setErrorState(ErrorState.CLOSE_CLEAN, null);
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("http11processor.request.invalidScheme"));
            }
        }
    }

    // Validate the characters in the URI. %nn decoding will be checked at
    // the point of decoding.
    for (int i = uriBC.getStart(); i < uriBC.getEnd(); i++) {
        //校验uri的字符，对于一些不规范的uri，返回400错误码
        if (!httpParser.isAbsolutePathRelaxed(uriB[i])) {
            response.setStatus(400);
            setErrorState(ErrorState.CLOSE_CLEAN, null);
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("http11processor.request.invalidUri"));
            }
            break;
        }
    }

    // Input filter setup
    //获取读取数据的过滤器
    InputFilter[] inputFilters = inputBuffer.getFilters();

    // Parse transfer-encoding header
    if (http11) {
        //解析转换编码请求头属性
        MessageBytes transferEncodingValueMB = headers.getValue("transfer-encoding");
        if (transferEncodingValueMB != null) {
            String transferEncodingValue = transferEncodingValueMB.toString();
            // Parse the comma separated list. "identity" codings are ignored
            int startPos = 0;
            int commaPos = transferEncodingValue.indexOf(',');
            String encodingName = null;
            while (commaPos != -1) {
                //然后添加编码转换过滤器，在读取数据的时候会进行编码
                encodingName = transferEncodingValue.substring(startPos, commaPos);
                addInputFilter(inputFilters, encodingName);
                startPos = commaPos + 1;
                commaPos = transferEncodingValue.indexOf(',', startPos);
            }
            encodingName = transferEncodingValue.substring(startPos);
            addInputFilter(inputFilters, encodingName);
        }
    }

    // Parse content-length header
    //如果请求中携带了数据，比如post方法的表达，json，文件流等
    long contentLength = request.getContentLengthLong();
    if (contentLength >= 0) {
        if (contentDelimitation) {
            // contentDelimitation being true at this point indicates that
            // chunked encoding is being used but chunked encoding should
            // not be used with a content length. RFC 2616, section 4.4,
            // bullet 3 states Content-Length must be ignored in this case -
            // so remove it.
            headers.removeHeader("content-length");
            request.setContentLength(-1);
        } else {
            inputBuffer.addActiveFilter(inputFilters[Constants.IDENTITY_FILTER]);
            contentDelimitation = true;
        }
    }

    // Validate host name and extract port if present
    //解析主机和端口
    parseHost(hostValueMB);

    if (!contentDelimitation) {
        // If there's no content length
        // (broken HTTP/1.0 or HTTP/1.1), assume
        // the client is not broken and didn't send a body
        //设置void过滤器，将不读取到任何数据
        inputBuffer.addActiveFilter(inputFilters[Constants.VOID_FILTER]);
        contentDelimitation = true;
    }

    if (getErrorState().isError()) {
        getAdapter().log(request, response, 0);
    }
}
```


以下是时序图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cd6f12f59893f9e574bcdaa44accf123.png)

