---
title: RPC原理浅析
date: 2018-04-07 22:10:32
---

#### 引言
很久以前就想结合网上一些资源以及自己的实践总结一篇属于自己的RPC笔记了，因为懒+时间关系，直到现在才完成一个简要的记录。本文只是一个简单粗糙的笔记记录，将别人的东西结合自己的理解整理了一下，思路主要来自于dubbo作者之一梁飞的博客：[RPC框架几行代码就够了](http://javatar.iteye.com/blog/1123915)。

#### 概述
RPC（Remote Procedure Call）远程过程调用，概念就不再赘述了。个人的理解，不管是什么样的RPC框架，总体思路都是服务提供方暴露服务，消费方通过服务方提供的接口使用动态代理获取代理对象，然后调用代理对象的方法，代理对象在内部进行远程调用，获得计算结果。简要示意图如下图所示：

![RPC示意图](https://upload-images.jianshu.io/upload_images/3727888-7587bbfb5ebb4c80.png)

在这个过程中，有两处关键点。第一是获取代理对象，第二是代理对象与服务提供方建立连接。对于获取代理对象的方式，需要了解Java的动态代理，可参考[Java的三种代理模式](https://www.cnblogs.com/cenyu/p/6289209.html)。总的来说，Java的动态代理（此处只指JDK中生成代理对象的API，不包括cglib）把建立远程连接的细节封装起来，使服务消费方可以在仅已知服务提供方的接口的情况下，可以像调用本地对象的方法一样去调用远程服务。

#### 简单实例
本实例使用socket建立连接，JDK的API做动态代理，主要有服务提供方暴露服务、服务消费方获取代理对象、代理对象与服务提供方建立远程连接并调用三个方面。忽略所有的参数校验、异常。
##### 暴露服务
已知的某个服务的实例对象service，建立ServerSocket服务，并监听指定端口，当有远程连接建立时，创建一个线程，在线程中从输入流中依次读取方法名、参数类型、参数值等信息，并根据方法名和参数执行实例对象service中对应的方法，获得返回结果。
```Java
public class RpcExport {
    private static int port = 1234;
    /**
     * 暴露服务
     * 
     * @param service
     *            服务的实现
     * @param port
     *            服务的端口
     * @throws Exception
     */
    public static void export(final Object service) throws Exception {

        System.out.println("Export service " + service.getClass().getName() + " on port " + port);

        // 创建socket，开始监听
        ServerSocket server = new ServerSocket(port);
        while (true) {
            final Socket socket = server.accept();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    ObjectInputStream input = null;
                    ObjectOutputStream output = null;
                    try {
                        // 从监听的socket中获得输入流
                        input = new ObjectInputStream(socket.getInputStream());
                        String methodName = input.readUTF();
                        Class<?>[] parameterTypes = (Class<?>[]) input.readObject();
                        Object[] arguments = (Object[]) input.readObject();
                        // 从监听的socket中获得输出流
                        output = new ObjectOutputStream(socket.getOutputStream());
                        Method method = service.getClass().getMethod(methodName, parameterTypes);
                        Object result = method.invoke(service, arguments);
                        output.writeObject(result);
                    } catch (Exception e) {
                    } finally {
                        try {
                            if (output != null) {
                                output.close();
                            }
                            if (input != null) {
                                input.close();
                            }
                            socket.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }).start();
        }
    }
}
```
##### 代理工厂
消费方获取代理对象，使用JDK的动态代理API，传入接口类名。生成代理对象的时候，需要传入一个实现了InvocationHandler的对象，也就是下面的RpcHandler，
```Java
import java.lang.reflect.Proxy;

public class RpcProxyFactory {

    public static <T> T getService(Class<T> interfaceClass) {

        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class[] { interfaceClass },
                 new RpcHandler());
    }

}
```
##### 代理对象建立远程连接
这个RpcHandler中重写了invoke方法，就是代理对象的方法执行的时候真正与服务提供方建立连接并获得返回结果的地方。
```Java
public class RpcHandler implements InvocationHandler {

    private String host = "127.0.0.1";

    private int port = 1234;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //与服务提供方建立连接
        Socket socket = new Socket(host, port);
        try {
            //获取输出流，并写出方法名、参数名、参数值
            ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream());
            try {
                output.writeUTF(method.getName());
                output.writeObject(method.getParameterTypes());
                output.writeObject(args);
                ObjectInputStream input = new ObjectInputStream(socket.getInputStream());
                try {
                    //获得返回结果
                    Object result = input.readObject();
                    if (result instanceof Throwable) {
                        throw (Throwable) result;
                    }
                    return result;
                } finally {
                    input.close();
                }
            } finally {
                output.close();
            }
        } finally {
            socket.close();
        }
    }
}
```
#### 使用实例
##### 服务接口
```Java
public interface HelloService {
    public String sayHello(String name);
}
```
##### 服务实现类
```Java
public class HelloServiceImpl implements HelloService{
    @Override
    public String sayHello(String name) {
        return "hello "+name;
    }
}
```
##### 暴露服务
```Java
public class RpcProvider {
    public static void main(String[] args) throws Exception {
        HelloService service = new HelloServiceImpl();
        RpcExport.export(service);
    }
}
```
##### 消费者获取代理对象并调用
```Java
public class RpcConsumer {
    public static void main(String[] args) throws Exception {
        HelloService proxy = RpcProxyFactory.getService(HelloService.class);
        String result = proxy.sayHello("world");
        System.out.println(result);
    }
}
```
##### 结果
输出：hello world

#### 总结
这个简易实例中，整个RPC原理很清晰，上面用例中最核心的一点，就是当proxy.sayHello执行的时候，实际是在执行RpcHandler的invoke方法，也就是远程调用。
如果要真正实现一个企业级RPC框架，仅仅有这个原理还是不够的。还需要考虑很多东西，例如建立连接的时候，使用NIO从而使得IO效率更高；或者在集群中，暴露服务的ip和端口都是动态的，而消费者此时也不能将要调用的服务提供方的ip和端口写死，于是需要一个注册中心的角色，产生注册服务、订阅服务等事件。
