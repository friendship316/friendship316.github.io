---
title: NIO单一长连接——dubbo通信模型实现
date: 2018-07-15
---
### 前言
前一段时间看了下dubbo，原想将dubbo详细总结下来，从使用简介、SPI扩展机制、Spring的scheme扩展、启动过程、动态注册与发现、分层设计、通信设计、线程模型等方面来总结，但是越看越发现架子太大，涉及的点太广，反而RPC的思想其实已经印象深刻了，再来总结这么多的点似乎不太值得，因为不懂的东西才是最有价值的，所以有了本文，将个人认为dubbo中比较有特色的通信模型总结于此，本文是一个demo，当然不乏一些脑补的东西在里面，如您偶然阅读此文发现问题，还请不吝指出问题所在。

### BIO通信缺陷
为了相对更好的理解dubbo这个通信模型的优势，首先需要回顾一下BIO通信的缺陷。
为此引用一下我之前借鉴dubbo作者梁飞的博客（[RPC框架几行代码就够了](http://javatar.iteye.com/blog/1123915)）而实现的一个用BIO实现的RPC示例：[RPC原理简析](http://friendship316.github.io/2018/04/07/rpcSimpleRecord/)，这里使用的是BIO的socket和ServerSocket实现通讯，用流来写入写出数据。BIO的完整连接示意图如下：

![完整BIO连接示意图](http://upload-images.jianshu.io/upload_images/3727888-76b1222bc2d3a0c3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 缺陷一、IO阻塞
可以看出，在BIO中，除了建立连接比较耗时之外，在客户端将数据传输到服务端之前，服务端的IO（输入流）阻塞，然后在服务端将返回值传输回来之前，客户端的IO（输入流）阻塞。也就是说**在一次RPC调用开始到完成之前，这个连接一直被此次调用所占用，但是实际上这次调用中，真正需要网络连接的只有中间的数据传输过程，在客户端写出和服务端读取并执行远端方法这两个时间点，其实网络连接是空闲的**。这就是BIO连接中浪费了网络资源的地方。

#### 缺陷二、大量连接
由于BIO的IO阻塞，导致每次RPC调用会占用一个连接。而正因为如此，为了减少频繁创建连接消耗的时间，引入了连接池（此处的连接池指普通的HTTP连接池，非异步连接池）的概念，连接池解决了频繁创建连接的资源消耗，但是没有解决根本性的阻塞问题。而且在服务消费者（客户端）数量远大于服务提供者（服务端）数量的时候，会导致服务提供者建立了大量的连接，而本身由于硬件资源的限制，单机最大连接数是有限的（这个限制以前是1w，也就是以前的C10K问题，据说近几年已经提升至50万个连接），所以在服务消费者过多，而服务提供者数量过少的情况下，服务提供者有因为过多的连接而被拖垮的风险（这需要极大的并发数，每秒上百万次的调用）。当然，要解决这个问题，增加机器，从而增加服务提供者数量是可以解决的，但是没有充分利用单机性能。

建立大量连接的另一个弊端，是操作系统频繁的线程上下文切换，因为连接数过多，线程切换频繁，会消耗大量的资源，而且这些切换可能不是必要的。比如当前建立了大量的连接，可能大部分处于阻塞状态，根本没有挨个挨个切换的必要。但是因为操作系统任务调度时并不会忽略阻塞状态的线程，所以造成浪费。

### NIO单一长连接实现分析

#### 长连接宏观简介
长连接其实不算NIO这里的特点，因为BIO也可以实现长连接（每次写完数据之后手动写入结束符而不关闭流就可以了），而且连接池一般也是使用的长连接方式。

NIO真正解决的是阻塞问题，因为阻塞问题解决了，所以也就不需要大量连接了。由于篇幅问题，此处不讨论NIO的细节（API），从相对宏观的角度介绍大体角色和调用方式，NIO三种角色示意如下图：

![NIO角色示意图](http://upload-images.jianshu.io/upload_images/3727888-2632e469660f2db2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

NIO由三种角色组成，Selector、SocketChannel、Buffer。

**SocketChannel**相当于BIO中的Socket，分为SocketChannel和ServerSocketChannel两种，是真正建立连接并传输数据的管道。这个管道不同于BIO的Socket的点就是，这个管道可以被多个线程共用，线程A使用这个管道写出数据了之后，线程B还可以使用这个管道写出数据，不再被某一次调用所独占。所以就可以不需要像BIO一样建立那么多的连接，一个客户端的一个连接就够了（当然，实际应用中因为机器都是多核，实际上建立核数个连接个人感觉是比较好的）。

**Buffer**是用来与SocketChannel互通数据的对象，本质上是一块内存区域。SocketChannel是不支持直接读写数据的，所有的读写操作必须通过Buffer来实现。值得一提的是，我们经常说在JVM中有的时候会使用虚拟机之外的内存，说的就是NIO中的Buffer，在分配内存的时候可以选择使用虚拟机外内存，减少数据的复制。

**Selector**是用来监控SocketChannel的事件的，其实是实现非阻塞的关键。NIO是基于事件的，将BIO中的流式传输改为了事件机制。BIO中，一个连接拥有一个输入／输出流，只要数据传输完成，流就可以读取数据。在NIO中，Selector定义了四种事件，OP_READ、OP_WRITE、OP_CONNECT、OP_ACCEPT。当服务端或者客户端收到写入完成的一次数据时，会触发OP_READ事件，此时可以从连接中读取数据。同理，当可以往连接中写入数据的时候，触发OP_WRITE事件（但是一般情况下这个事件没有必要，因为连接一般都是可写的）。客户端与服务端建立连接的时候，客户端会收到OP_CONNECT事件，而服务端会触发OP_ACCEPT事件。通过这一系列事件将数据的发送与读写解耦，实现异步调用。将一个SocketChannel+一个事件绑定在一个Selector上，Selector本质上是轮询每一个SocketChannel，如果没有事件触发，那么线程阻塞，如果有事件触发，返回对应的SocketChannel，以便进行后续的处理。

#### 使用NIO设计RPC调用分析

前面提到，由于NIO的SocketChannel是非阻塞的，所以不再需要连接池，使用一个连接就够了。

![NIO单一长连接RPC线程模型](http://upload-images.jianshu.io/upload_images/3727888-2a3e097144cd7a11?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是如果真的使用NIO来进行RPC调用的话，会有数据和调用方对应不上的问题，如下图：

![NIO异步顺序问题](http://upload-images.jianshu.io/upload_images/3727888-a93021a1bbf552f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图所示，如果多个线程共用一个连接，那么每个线程调用之后返回的顺序是不可控的，所以有可能先发出数据的反而后得到返回值，这就使得数据对应不上了。个人觉得因为这一点，NIO及其适合聊天室类型的设计，因为每个聊天方都是一个单独的SocketChannel连接，而此时并没有顺序问题。

但是对RPC调用来说，每次调用的返回值必须与调用方对应上，为此，Dubbo的设计是给每个请求设计一个请求id，在发送请求与发送返回值时都带上这个id。详细思路如下图：

![NIO单一长连接的RPC设计](http://upload-images.jianshu.io/upload_images/3727888-098e4b3fb2b736f2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

业务线程在发出请求之前，需要存储一个请求对象，同时挂起相应的业务线程（挂起不会被任务调度，所以不存在线程切换消耗），这个请求对象包含了此次请求的id，然后在获取服务端返回的数据的时候，解析出这个id，通过这个id取出请求对象，并唤醒对应的线程。

### NIO单一长连接实现demo

废话了这么多，终于可以上代码了，我这里服务端使用了线程池去执行远端方法，使得真正的服务端线程只需要读取数据就可以了。可能作为demo来讲，写的有些复杂，但是理解的时候以RpcNioConsumer和RpcNioProvider作为入口就比较好梳理了。NIO的主要实现在RpcNioMultClient和RpcNioMultServer中。

#### 工具类

序列化／反序列化工具类
```Java
package com.liepin.common;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

/**
 * @author: lifs
 * @create: 2018-06-27 22:03
 **/
public class SerializeUtil {

    public static byte[] serialize(Object obj) {
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(bos);
            oos.writeObject(obj);
            oos.flush();
            byte[] bytes = bos.toByteArray();
            return bytes;
        } catch (IOException e) {
            System.out.println("序列化对象出错！");
            e.printStackTrace();
            return null;
        }
    }

    public static Object unSerialize(byte[] bytes) {
        try {
            ByteArrayInputStream bis = new ByteArrayInputStream(bytes);
            ObjectInputStream ois = new ObjectInputStream(bis);
            return ois.readObject();
        } catch (IOException e) {
            System.out.println("反序列化出错！");
            e.printStackTrace();
            return null;
        } catch (ClassNotFoundException e) {
            System.out.println("反序列化出错！");
            e.printStackTrace();
            return null;
        }
    }
}
```

#### 服务端代码

RPC调用的接口：
```Java
package com.liepin.service;

/**
 * @author: lifs
 * @create: 2018-04-01 20:35
 **/
public interface HelloService {
    public String sayHello(String name);
}
```
服务端的接口实现：
```Java
package com.liepin.service;
/**
 * @author: lifs
 * @create: 2018-04-01 20:40
 **/
public class HelloServiceImpl implements HelloService {
    @Override
    public String sayHello(String name) {
        return "hello " + name;
    }
}
```
服务端，用于与客户端建立连接、读取数据：
```Java
package com.liepin.niomultipart;

import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;

import com.liepin.common.BeanContainer;
import com.liepin.common.RequstMultObject;
import com.liepin.common.SerializeUtil;
import com.liepin.common.ThreadPoolUtil;

/**
 * @author: lifs
 * @create: 2018-07-01 11:38
 **/
public class RpcNioMultServer {

    // 通道管理器
    private Selector selector;

    public static void start() throws IOException {
        RpcNioMultServer server = new RpcNioMultServer();
        server.initServer(8080);
        server.listen();
    }

    /**
     * 获得一个ServerSocket通道，并对该通道做一些初始化的工作
     *
     * @param port
     *            绑定的端口号
     * @throws IOException
     */
    public void initServer(int port) throws IOException {
        // 获得一个ServerSocket通道
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        // 设置通道为非阻塞
        serverChannel.configureBlocking(false);
        // 将该通道对应的ServerSocket绑定到port端口
        serverChannel.socket().bind(new InetSocketAddress(port));
        // 获得一个通道管理器
        this.selector = Selector.open();
        // 将通道管理器和该通道绑定，并为该通道注册SelectionKey.OP_ACCEPT事件,注册该事件后，
        // 当该事件到达时，selector.select()会返回，如果该事件没到达selector.select()会一直阻塞。
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
    }

    public void listen() {
        System.out.println("服务端启动成功！");
        // 轮询访问selector
        while (true) {
            try {
                // 当注册的事件到达时，方法返回；否则,该方法会一直阻塞
                selector.select();
                // 获得selector中选中的项的迭代器，选中的项为注册的事件
                Iterator ite = selector.selectedKeys().iterator();
                while (ite.hasNext()) {
                    SelectionKey key = (SelectionKey) ite.next();
                    // 删除已选的key,以防重复处理
                    ite.remove();
                    // 客户端请求连接事件
                    if (key.isAcceptable()) {
                        ServerSocketChannel server = (ServerSocketChannel) key.channel();
                        // 获得和客户端连接的通道
                        SocketChannel channel = server.accept();
                        // 设置成非阻塞
                        channel.configureBlocking(false);

                        // 在和客户端连接成功之后，为了可以接收到客户端的信息，需要给通道设置读的权限。
                        channel.register(this.selector, SelectionKey.OP_READ);

                        // 获得了可读的事件
                    } else if (key.isReadable()) {
                        SocketChannel channel = (SocketChannel) key.channel();
                        byte[] bytes = readMsgFromClient(channel);
                        if (bytes != null && bytes.length > 0) {
                            // 读取之后将任务放入线程池异步返回
                            RpcNioMultServerTask task = new RpcNioMultServerTask(bytes, channel);
                            ThreadPoolUtil.addTask(task);
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
    }

    public byte[] readMsgFromClient(SocketChannel channel) {
        ByteBuffer byteBuffer = ByteBuffer.allocate(4);
        try {
            // 首先读取消息头（自己设计的协议头，此处是消息体的长度）
            int headCount = channel.read(byteBuffer);
            if (headCount < 0) {
                return null;
            }
            byteBuffer.flip();
            int length = byteBuffer.getInt();
            // 读取消息体
            byteBuffer = ByteBuffer.allocate(length);
            int bodyCount = channel.read(byteBuffer);
            if (bodyCount < 0) {
                return null;
            }
            return byteBuffer.array();
        } catch (IOException e) {
            System.out.println("读取数据异常");
            e.printStackTrace();
            return null;
        }
    }
}

```
线程池工具类
```Java
package com.liepin.common;

import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

import com.liepin.niomultipart.RpcNioMultServerTask;

/**
 * @author: lifs
 * @create: 2018-07-01 21:39
 **/
public class ThreadPoolUtil {

    private static ThreadPoolExecutor executor;

    public static void init() {
        if (executor == null) {
            synchronized (ThreadPoolUtil.class) {
                executor = new ThreadPoolExecutor(10, 20, 200, TimeUnit.MILLISECONDS, new LinkedBlockingDeque<>());
            }
        }
    }

    public static void addTask(RpcNioMultServerTask task) {
        if (executor == null) {
            init();
        }
        executor.execute(task);
    }
}

```
用于获取数据执行远端方法的线程池任务类：
```Java
package com.liepin.niomultipart;

import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

import com.liepin.common.BeanContainer;
import com.liepin.common.RequstMultObject;
import com.liepin.common.SerializeUtil;

/**
 * 服务端线程池任务
 * 
 * @author: lifs
 * @create: 2018-07-01 21:18
 **/
public class RpcNioMultServerTask implements Runnable {

    private byte[] bytes;

    private SocketChannel channel;

    public RpcNioMultServerTask(byte[] bytes, SocketChannel channel) {
        this.bytes = bytes;
        this.channel = channel;
    }

    @Override
    public void run() {
        if (bytes != null && bytes.length > 0 && channel != null) {
            // 反序列化
            RequstMultObject requstMultObject = (RequstMultObject) SerializeUtil.unSerialize(bytes);
            // 调用服务并序列化结果然后返回
            requestHandle(requstMultObject, channel);
        }
    }

    public void requestHandle(RequstMultObject requstObject, SocketChannel channel) {
        Long requestId = requstObject.getRequestId();
        Object obj = BeanContainer.getBean(requstObject.getCalzz());
        String methodName = requstObject.getMethodName();
        Class<?>[] parameterTypes = requstObject.getParamTypes();
        Object[] arguments = requstObject.getArgs();
        try {
            Method method = obj.getClass().getMethod(methodName, parameterTypes);
            String result = (String) method.invoke(obj, arguments);
            byte[] bytes = SerializeUtil.serialize(result);
            ByteBuffer buffer = ByteBuffer.allocate(bytes.length + 12);
            // 为了便于客户端获得请求ID，直接将id写在头部（这样客户端直接解析即可获得，不需要将所有消息反序列化才能得到）
            // 然后写入消息题的长度，最后写入返回内容
            buffer.putLong(requestId);
            buffer.putInt(bytes.length);
            buffer.put(bytes);
            buffer.flip();
            channel.write(buffer);
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException | IOException e) {
            e.printStackTrace();
        }
    }

    public SocketChannel getChannel() {
        return channel;
    }

    public void setChannel(SocketChannel channel) {
        this.channel = channel;
    }
}

```
服务端服务发布：
```Java
package com.liepin.rpc;

import java.io.IOException;

import com.liepin.common.BeanContainer;
import com.liepin.niomultipart.RpcNioMultServer;
import com.liepin.service.HelloService;
import com.liepin.service.HelloServiceImpl;

/**
 * @author: lifs
 * @create: 2018-06-28 23:24
 **/
public class RpcNioProvider {
    public static void main(String[] args) throws IOException {
        // 将服务放进bean容器
        HelloService helloService = new HelloServiceImpl();
        BeanContainer.addBean(HelloService.class, helloService);
        // 启动NIO服务端
        startMultRpcNioServer();
    }

    public static void startMultRpcNioServer() {
        Runnable r = () -> {
            try {
                RpcNioMultServer.start();
            } catch (IOException e) {
                e.printStackTrace();
            }
        };
        Thread t = new Thread(r);
        t.start();
    }
}
```
服务端服务发布时使用的bean容器：
```Java
package com.liepin.common;

import java.util.concurrent.ConcurrentHashMap;

/**
 * @author: lifs
 * @create: 2018-06-28 07:52
 **/
public class BeanContainer {

    private static ConcurrentHashMap<Class<?>, Object> container = new ConcurrentHashMap<>();

    public static boolean addBean(Class<?> clazz, Object object) {
        container.put(clazz, object);
        return true;
    }

    public static Object getBean(Class<?> clazz) {
        return container.get(clazz);
    }
}
```

#### 客户端代码

客户端：
```Java
package com.liepin.niomultipart;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Iterator;

import com.liepin.common.RpcContainer;

/**
 * @author: lifs
 * @create: 2018-07-01 01:12
 **/
public class RpcNioMultClient {

    private static RpcNioMultClient rpcNioClient;

    // 通道管理器
    private Selector selector;

    // 通道
    private SocketChannel channel;

    private String serverIp = "localhost";

    private int port = 8080;

    private RpcNioMultClient() {
        // 初始化client
        initClient();
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                listen();
            }
        };
        Thread t = new Thread(runnable);
        t.start();
    }

    public static RpcNioMultClient getInstance() {
        if (rpcNioClient == null) {
            synchronized (RpcNioMultClient.class) {
                if (rpcNioClient == null) {
                    rpcNioClient = new RpcNioMultClient();
                }
            }
        }
        return rpcNioClient;
    }

    public void initClient() {
        try {
            // 打开一个通道
            channel = SocketChannel.open();
            // 设置为非阻塞通道（异步）
            channel.configureBlocking(false);
            // 获得通道管理器，用于监听通道事件
            selector = Selector.open();
            // 建立连接
            channel.connect(new InetSocketAddress(serverIp, port));
            // 由于是非阻塞的，所以有可能连接并未建立完成，调用finishConnect完成连接
            if (channel.isConnectionPending()) {
                channel.finishConnect();
            }
            System.out.println("客户端初始化完成，建立连接完成");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void listen() {
        try {
            while (true) {
                // 绑定到通道管理器,监听可读事件，因为客户端只需要从服务端获得数据然后读取，所以只需要监听READ事件
                channel.register(selector, SelectionKey.OP_READ);
                // 开始轮询READ事件
                selector.select();
                Iterator ite = selector.selectedKeys().iterator();
                while (ite.hasNext()) {
                    SelectionKey key = (SelectionKey) ite.next();
                    // 删除已选的key,以防重复处理
                    ite.remove();
                    if (key.isReadable()) {
                        // 读取信息
                        readMsgFromServer();
                    }
                }
            }
        } catch (IOException e) {
            System.out.println("客户端建立连接失败");
        }
    }

    public boolean sendMsg2Server(byte[] bytes) {
        try {
            ByteBuffer buffer = ByteBuffer.allocate(bytes.length + 4);
            // 放入消息长度，然后放入消息体
            buffer.putInt(bytes.length);
            buffer.put(bytes);
            // 写完之后buffer设置为可读状态
            buffer.flip();
            // 写出消息
            channel.write(buffer);
        } catch (IOException e) {
            System.out.println("客户端写出消息失败！");
            e.printStackTrace();
        }
        return true;
    }

    public void readMsgFromServer() {
        ByteBuffer byteBuffer;
        try {
            // 首先读取请求id
            byteBuffer = ByteBuffer.allocate(8);
            int readIdCount = channel.read(byteBuffer);
            if (readIdCount < 0) {
                return;
            }
            byteBuffer.flip();
            Long requsetId = byteBuffer.getLong();

            // 读取返回值长度
            byteBuffer = ByteBuffer.allocate(4);
            int readHeadCount = channel.read(byteBuffer);
            if (readHeadCount < 0) {
                return;
            }
            // 将buffer切换为待读取状态
            byteBuffer.flip();
            int length = byteBuffer.getInt();

            // 读取消息体
            byteBuffer = ByteBuffer.allocate(length);
            int readBodyCount = channel.read(byteBuffer);
            if (readBodyCount < 0) {
                return;
            }
            byte[] bytes = byteBuffer.array();

            // 将返回值放入指定容器
            RpcContainer.addResponse(requsetId, bytes);
        } catch (IOException e) {
            System.out.println("读取数据异常");
            e.printStackTrace();
        }
    }
}
```
请求时的数据结构
```
package com.liepin.common;

import java.io.Serializable;

/**
 * @author: lifs
 * @create: 2018-07-01 11:10
 **/
public class RequstMultObject implements Serializable {

    private static final long serialVersionUID = 3132836600205356306L;

    // 请求id
    private Long requestId;

    // 服务提供者接口
    private Class<?> calzz;

    // 服务的方法名称
    private String methodName;

    // 参数类型
    private Class<?>[] paramTypes;

    // 参数
    private Object[] args;

    public RequstMultObject(Class<?> calzz, String methodName, Class<?>[] paramTypes, Object[] args) {
        this.calzz = calzz;
        this.methodName = methodName;
        this.paramTypes = paramTypes;
        this.args = args;
    }

    public Long getRequestId() {
        return requestId;
    }

    public void setRequestId(Long requstId) {
        this.requestId = requstId;
    }

    public Class<?> getCalzz() {
        return calzz;
    }

    public void setCalzz(Class<?> calzz) {
        this.calzz = calzz;
    }

    public String getMethodName() {
        return methodName;
    }

    public void setMethodName(String methodName) {
        this.methodName = methodName;
    }

    public Class<?>[] getParamTypes() {
        return paramTypes;
    }

    public void setParamTypes(Class<?>[] paramTypes) {
        this.paramTypes = paramTypes;
    }

    public Object[] getArgs() {
        return args;
    }

    public void setArgs(Object[] args) {
        this.args = args;
    }
}

```
请求数据获取类，用来挂起和唤醒挂起的线程：
```Java
package com.liepin.common;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author: lifs
 * @create: 2018-07-01 16:53
 **/
public class RpcResponseFuture {

    private final Lock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    private Long requstId;

    public RpcResponseFuture(Long requstId) {
        this.requstId = requstId;
    }

    public byte[] get() {
        byte[] bytes = RpcContainer.getResponse(requstId);
        if (bytes == null || bytes.length < 0) {
            lock.lock();
            try {
                System.out.println("请求id:" + requstId + ",请求结果尚未返回，线程挂起");
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
        System.out.println("请求id:" + requstId + ",请求结果返回，线程挂起结束");
        return RpcContainer.getResponse(requstId);
    }

    public void rpcIsDone() {
        lock.lock();
        try {
            condition.signal();
        } finally {
            lock.unlock();
        }
    }

    public Long getRequstId() {
        return requstId;
    }

    public void setRequstId(Long requstId) {
        this.requstId = requstId;
    }
}

```
RPC容器 用来存储发送RPC请求时的请求对象，以及存储返回值
```Java
package com.liepin.common;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * @author: lifs
 * @create: 2018-07-01 10:58
 **/
public class RpcContainer {
    //返回值容器
    private static ConcurrentHashMap<Long, byte[]> responseContainer = new ConcurrentHashMap<>();
    //请求对象容器
    private static ConcurrentHashMap<Long, RpcResponseFuture> requestFuture = new ConcurrentHashMap<>();
    //请求id
    private volatile static AtomicLong requstId = new AtomicLong(0);

    public static Long getRequestId() {
        return requstId.getAndIncrement();
    }

    public static void addResponse(Long requestId, byte[] responseBytes) {
        responseContainer.put(requestId, responseBytes);
        RpcResponseFuture responseFuture = requestFuture.get(requestId);
        responseFuture.rpcIsDone();
    }

    public static byte[] getResponse(Long requestId) {
        return responseContainer.get(requestId);
    }

    public static void addRequstFuture(RpcResponseFuture rpcResponseFuture) {
        requestFuture.put(rpcResponseFuture.getRequstId(), rpcResponseFuture);
    }

    public static RpcResponseFuture getRpcRequstFutue(Long requestId) {
        return requestFuture.get(requestId);
    }

    public static void removeResponseAndFuture(Long requestId) {
        responseContainer.remove(requestId);
        requestFuture.remove(requestId);
    }
}

```
RPC代理工厂类：
```Java
package com.liepin.proxy;

import java.lang.reflect.Proxy;

import com.liepin.niomultipart.RpcNIoMultHandler;
/**
 * @author: lifs
 * @create: 2018-06-28 23:35
 **/
public class RpcProxyFactory {
    /**
     * 多线程环境代理对象
     * 
     * @param interfaceClass
     * @return T
     * @throws BizException
     * @createTime：2018/7/1
     * @author: shakeli
     */
    public static <T> T getMultService(Class<T> interfaceClass) {
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class[] { interfaceClass },
                new RpcNIoMultHandler());
    }
}
```
实际代理类
```Java
package com.liepin.niomultipart;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

import com.liepin.common.RequstMultObject;
import com.liepin.common.RpcContainer;
import com.liepin.common.RpcResponseFuture;
import com.liepin.common.SerializeUtil;

/**
 * @author: lifs
 * @create: 2018-07-01 11:41
 **/
public class RpcNIoMultHandler implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 获得请求id
        Long responseId = RpcContainer.getRequestId();
        // 封装请求对象
        RequstMultObject requstMultObject = new RequstMultObject(method.getDeclaringClass(), method.getName(),
                method.getParameterTypes(), args);
        requstMultObject.setRequestId(responseId);

        // 封装设置rpcResponseFuture，主要用于获取返回值
        RpcResponseFuture rpcResponseFuture = new RpcResponseFuture(responseId);
        RpcContainer.addRequstFuture(rpcResponseFuture);

        // 序列化
        byte[] requstBytes = SerializeUtil.serialize(requstMultObject);
        // 发送请求信息
        RpcNioMultClient rpcNioMultClient = RpcNioMultClient.getInstance();
        rpcNioMultClient.sendMsg2Server(requstBytes);

        // 从ResponseContainer获取返回值
        byte[] responseBytes = rpcResponseFuture.get();
        if (requstBytes != null) {
            RpcContainer.removeResponseAndFuture(responseId);
        }

        // 反序列化获得结果
        Object result = SerializeUtil.unSerialize(responseBytes);
        System.out.println("请求id：" + responseId + " 结果：" + result);
        return result;
    }
}
```
客户端RPC调用：
```Java
package com.liepin.rpc;

import com.liepin.proxy.RpcProxyFactory;
import com.liepin.service.HelloService;

/**
 * @author: lifs
 * @create: 2018-06-28 23:34
 **/
public class RpcNioConsumer {
    public static void main(String[] args) {
        multipartRpcNio();
    }

    /**
     * 多线程IO调用示例
     * 
     * @param
     * @return void
     * @throws BizException
     * @createTime：2018/7/1
     * @author: shakeli
     */
    public static void multipartRpcNio() {
        HelloService proxy = RpcProxyFactory.getMultService(HelloService.class);
        for (int i = 0; i < 100; i++) {
            final int j = I;
            Runnable runnable = new Runnable() {
                @Override
                public void run() {
                    String result = proxy.sayHello("world!");
                }
            };
            Thread t = new Thread(runnable);
            t.start();
        }
    }
}
```