---
title: dubbo使用入门-使用zookeeper作为注册中心
date: 2018-06-08 15:38:24
---
关于dubbo的使用，在dubbo的apache官网上其实说的很清楚了，在这里只是记录一下dubbo使用zookeeper作为注册中心的使用例子。使用的是zookeeper伪集群配置，关于zookeeper伪集群配置的安装请参考[zookeeper伪集群安装](https://www.jianshu.com/p/f13a17e3e351)。
### 工程目录结构
这里创建三个maven工程（此处不是web工程），dubbo-consumer作为服务消费方，dubbo-provider作为服务提供方，dubbo-interface作为公共jar包提供引用接口。如下图所示，dubbo-interface中只定义了一个接口HelloService，有一个方法sayHello。

![工程目录结构](http://p9bex2mzr.bkt.clouddn.com/dubbo_projects.png)

### 服务提供方
dubbo-provider中的pom.xml加入如下依赖：
```xml
<!--dubbo-interface 接口依赖-->
<dependency>
    <groupId>com.liepin</groupId>
    <artifactId>dubbo-interface</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
<!--dubbo依赖-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo</artifactId>
    <version>2.6.1</version>
</dependency>
<!--zookeeper-->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.9</version>
    <type>pom</type>
</dependency>
<!--zkClient-->
<dependency>
    <groupId>com.github.sgroschupf</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.1</version>
</dependency>
```
服务提供方dubbo-provider中的HelloServiceImpl是HelloService接口的实现类。
```Java
package com.liepin.demo;
import com.liepin.interfaces.HelloService;

/**
 * @author: lifs
 * @create: 2018-05-22 08:05
 **/
public class HelloServiceImpl implements HelloService {
    @Override
    public String sayHello(String name) {
        return "hello!"+name;
    }
}
```
配置provider.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <!--定义了提供方应用信息，用于计算依赖关系；在 dubbo-admin 或 dubbo-monitor 会显示这个名字，方便辨识-->
    <dubbo:application name="demo-provider"/>
    <!--使用 zookeeper 注册中心暴露服务，要先开启 zookeeper,可配置集群和多组注册中心-->
    <dubbo:registry protocol="zookeeper" address="127.0.0.1:2182,127.0.0.1:2183,127.0.0.1:2184" client="zkclient"/>
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880"/>
    <!--使用 dubbo 协议实现定义好的 HelloService 接口-->
    <dubbo:service interface="com.liepin.interfaces.HelloService" ref="demoService"/>
    <!--具体实现该接口的 bean-->
    <bean id="demoService" class="com.liepin.demo.HelloServiceImpl"/>
</beans>
```
dubbo所有的配置均可以使用注解方式，比如在所有服务上加上` @com.alibaba.dubbo.config.annotation.Service `这个注解以及spring bean的注解即可替代下面两行的XML配置。
```xml
    <!--使用 dubbo 协议实现定义好的 HelloService 接口-->
    <dubbo:service interface="com.liepin.interfaces.HelloService" ref="demoService"/>
    <!--具体实现该接口的 bean-->
    <bean id="demoService" class="com.liepin.demo.HelloServiceImpl"/>
```
启动服务提供方，由于使用的是spring的配置，所以此处实际上是启动了spring容器，在spring启动的时候去加载dubbo的配置并注册服务。
```Java
package com.liepin.demo;

import java.io.IOException;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * @author: lifs
 * @create: 2018-05-22 23:04
 **/
public class Provider {

    public static void main(String[] args) throws IOException {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(
                new String[] {"provider.xml"});
        context.start();
        System.out.println("服务提供者启动，zk端口2882,2883,2884");
        System.in.read();//阻塞，按任意键停止服务
    }
}
```
### 服务消费方
在dubbo-consumer的pom.xml中加入如下依赖：
```xml
<!--dubbo-interface 接口包-->
<dependency>
  <groupId>com.liepin</groupId>
  <artifactId>dubbo-interface</artifactId>
  <version>1.0-SNAPSHOT</version>
</dependency>
<!--dubbo依赖-->
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>dubbo</artifactId>
  <version>2.6.1</version>
</dependency>
<!--zookeeper-->
<dependency>
  <groupId>org.apache.zookeeper</groupId>
  <artifactId>zookeeper</artifactId>
  <version>3.4.12</version>
  <type>pom</type>
</dependency>
<!--zkClient-->
<dependency>
  <groupId>com.github.sgroschupf</groupId>
  <artifactId>zkclient</artifactId>
  <version>0.1</version>
</dependency>
```
配置consumer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <!--定义消费方-->
    <dubbo:application name="demo-consumer"/>
    <!--向 zookeeper 订阅 provider 的地址，由 zookeeper 定时推送-->
    <dubbo:registry protocol="zookeeper" address="127.0.0.1:2182,127.0.0.1:2183,127.0.0.1:2184" client="zkclient"/>
    <!--使用 dubbo 协议调用定义好的 HelloService 接口-->
    <dubbo:reference id="demoService" interface="com.liepin.interfaces.HelloService"/>
</beans>
```
由于也是使用的spring配置，所以启动spring容器：
```Java
package com.liepin.demo;

import org.springframework.context.support.ClassPathXmlApplicationContext;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
import com.liepin.interfaces.HelloService;

/**
 * @author: lifs
 * @create: 2018-05-22 22:42
 **/
public class Consumer {

    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(
                new String[] {"consumer.xml"});
        context.start();
        HelloService helloService = context.getBean(HelloService.class);
        System.out.println(helloService.sayHello("boy"));
    }
}
```
这里获取HelloService对象也可以使用注解` @com.alibaba.dubbo.config.annotation.Reference `。
调用服务之后输出“hello!boy”即调用成功。本例中工程非web工程。如果是web工程，直接在web.xml中加入spring的listener即可启动dubbo，无需额外添加dubbo的listener。如果不使用spring，dubbo还可以通过com.alibaba.dubbo.container.Main的main方法启动，不过考虑现在不用spring的Java项目实在太少，所以就不过多探讨。
### duboo使用zookeeper注册
众所周知，dubbo使用zookeeper的命名服务进行注册。这里进入zookeeper去看一看dubbo注册的znode是什么样的。
在服务提供方和服务消费方启动之前，进入zookeeper集群(三个节点简称为zoo1、zoo2、zoo3)中的第一个节点zoo1，可以看到只有默认的节点zookeeper。

![zookeeper默认节点](http://p9bex2mzr.bkt.clouddn.com/zookeeper_default_node.png)

启动provider之后：

![provider](http://p9bex2mzr.bkt.clouddn.com/zk_dubbo_znode.png)

可以看到在根结点下新增了节点dubbo，然后在dubbo节点下有了HelloService节点，HelloService下面有这个服务的配置节点configurators和提供方providers，由于目前我只启动了一个服务，所以providers下面只有一个节点。这个节点的内容是一个URL（这个URL包含了服务提供方的详细信息，详细含义此处不作解读），将这个URL转义之后可以得到：
>dubbo://192.168.0.103:20880/com.liepin.interfaces.HelloService?anyhost=true&application=demo-provider&dubbo=2.6.1&generic=false&interface=com.liepin.interfaces.HelloService&methods=sayHello&pid=2520&revision=1.0-SNAPSHOT&side=provider&timestamp=1528621287574

接下来我们再启动consumer（此处为了便于观察，我的consumer调用之后并未结束程序，而是在Consumer的main方法后面加上System.in.read()使其阻塞了）：

![consumer](http://p9bex2mzr.bkt.clouddn.com/zk_dubbo_consumer_znode.png)

可以看到，启动了consumer之后，HelloService节点下新增了两个节点：consumers和routers。consumers下面也是一个URL（这个URL包含了服务消费方的详细信息，详细含义此处不作解读）：

>consumer://192.168.0.103/com.liepin.interfaces.HelloService?application=demo-consumer&category=consumers&check=false&dubbo=2.6.1&interface=com.liepin.interfaces.HelloService&methods=sayHello&pid=2683&revision=1.0-SNAPSHOT&side=consumer&timestamp=1528622384726

使用get命令查看节点数据：

![查看节点数据](http://p9bex2mzr.bkt.clouddn.com/zk_dubbo_consumer_znode_info.png)

可以看到，内容为null，所以dubbo创建的这个节点并没有内容，只是使用了命名而已，所以称为命名服务。
值得一提的是，这个接口下的consumer和provider都是临时节点，当这个zk会话结束之后并且在zookeeper设置的session过期时间之后，这个节点会删除。
前面我使用System.in.read()让consumer阻塞了，接下来按任意键使其阻塞状态解除，可以看到consumers节点仍然存在，但是下面的消费者是空，说明没有任何消费者了。

![consumer临时节点](http://p9bex2mzr.bkt.clouddn.com/consumer_linshi_znode.png)

同样，如果取消provider的阻塞，停止服务，那么/dubbo/com.liepin.interfaces.HelloService/providers节点也不再有任何子节点。

