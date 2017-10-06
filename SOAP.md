---
title: 基于SOAP的webservice（1）、JAX-WS实现
date: 2017-05-14
---
* 因为工作中使用了SOAP进行两个系统的接口调用，所以私下学习一下两种实现，粗略记录于此。本文侧重于实际实现操作，而不是理论原理。
* 个人简单理解：SOAP（Simple Object Access Protocol 简单对象访问协议）是基于**XML**和**HTTP**的用于实现网络连通的程序之间远程调用的协议(但是SOAP1.2中也可以使用非HTTP协议进行传输)。两个通过网络连接的程序体，通过一定规范的XML进行远程调用。更详细一点的可以参考[浅谈 SOAP](https://www.ibm.com/developerworks/cn/xml/x-sisoap/)。

### 方式1、JAX-WS实现
JAX-WS是JDK自带的Web服务的API，它可以用于提供REST式或基于SOAP的服务。SOAP的JAX-WS实现有两种部署方式，一种是直接在应用程序中使用Endpoint发布；另一种是在web工程的web.xml中配置监听器，随着web工程的启动而启动。为了两种方式都使用，并且便于使用依赖，我这里创建的是一个maven web工程，工程名为SOAPServer，JDK版本是1.8。
#### 1\. 编写服务端。
用于提供服务的服务端类ServerDemo ：
```java
package demo;
import javax.jws.WebMethod;
import javax.jws.WebService;

@WebService
public class ServerDemo {
	
	@WebMethod
	public String sayHi(String name){
		return "Hi: "+name+",this is SOAPServer";
	}
	
	@WebMethod
	public String doSth(){
		return "do some thing";
	}
}
```
这个类很简单，只有两个方法，一个有参数，一个无参数。类名前面加了@WebService注解，方法名前面加了@WebMethod注解（SOAP还有许多其他的注解）。

#### 2\. 发布服务。
前面提到发布服务有两种方式。
##### （1）、使用Endpoint发布，编写用于发布的类：
```java
package demo;
import javax.xml.ws.Endpoint;

public class Publisher {
	public static void main(String[] args) {
		String url = "http://localhost:8080/fistSoap";
		Endpoint.publish(url, new ServerDemo());
	}
}

```
运行main方法，使用浏览器访问http://localhost:8080/fistSoap， 如果有如下所示，即说明发布正常。 ![访问服务](http://upload-images.jianshu.io/upload_images/3727888-86b0a48176c5fef5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
##### （2）、使用web项目监听器发布。
  首先需要添加依赖：
```xml
    <dependency>
		<groupId>com.sun.xml.ws</groupId>
		<artifactId>jaxws-rt</artifactId>
		<version>2.2.10</version>
	</dependency>
```
然后在WEB-INFO目录下添加名为sun-jaxws.xml（默认名称）的文件：
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<endpoints version="2.0" xmlns="http://java.sun.com/xml/ns/jax-ws/ri/runtime">  
  
  <endpoint name="firstSoap" implementation="demo.ServerDemo" url-pattern="/firstSoap"></endpoint>
    
</endpoints> 
```
然后在web.xml中添加监听器和servlet(此处url-pattern为/*，拦截所有请求)：
```xml
    <listener>  
	    <listener-class>com.sun.xml.ws.transport.http.servlet.WSServletContextListener</listener-class>  
	</listener> 
    <servlet>
    	<servlet-name>jaxws</servlet-name>
    	<servlet-class>com.sun.xml.ws.transport.http.servlet.WSServlet</servlet-class>
    	<load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
    	<servlet-name>jaxws</servlet-name>
    	<url-pattern>/*</url-pattern>
    </servlet-mapping>
```
然后在浏览器中访问http://localhost:8080/SOAPServer/firstSoap， 出现与Endpoint方式一样的web服务的概述页面，说明发布正常。如果有多个服务，那么只需要在sun-jaxws.xml中增加一条endpoint即可。例如我在sun-jaxws.xml增加了一个如下服务，那么我访问http://localhost:8080/SOAPServer/firstSoap或者 http://localhost:8080/SOAPServer/randService 会显示两个服务的信息。
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<endpoints version="2.0" xmlns="http://java.sun.com/xml/ns/jax-ws/ri/runtime">  
  
  <endpoint name="firstSoap" implementation="demo.ServerDemo" url-pattern="/firstSoap"></endpoint>
  <endpoint name="RandService" implementation="rand.RandService" url-pattern="/randService" />  
    
</endpoints>  
```
![多个服务信息](http://upload-images.jianshu.io/upload_images/3727888-706c231575c833bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3\. 生成客户端代码并调用。
服务发布后使用浏览器访问http://localhost:8080/SOAPServer/hello?wsdl 即可获得该服务的wsdl文档——描述服务调用信息（wsdl文档的详细结构在后续结合XML报文格式进行分析）：
```xml
	<definitions
    xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd"
    xmlns:wsp="http://www.w3.org/ns/ws-policy"
    xmlns:wsp1_2="http://schemas.xmlsoap.org/ws/2004/09/policy"
    xmlns:wsam="http://www.w3.org/2007/05/addressing/metadata"
    xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
    xmlns:tns="http://demo/"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns="http://schemas.xmlsoap.org/wsdl/" targetNamespace="http://demo/" name="ServerDemoService">
    <types>
        <xsd:schema>
            <xsd:import namespace="http://demo/" schemaLocation="http://localhost:8080/SOAPServer/firstSoap?xsd=1"/>
        </xsd:schema>
    </types>
    <message name="sayHi">
        <part name="parameters" element="tns:sayHi"/>
    </message>
    <message name="sayHiResponse">
        <part name="parameters" element="tns:sayHiResponse"/>
    </message>
    <message name="doSth">
        <part name="parameters" element="tns:doSth"/>
    </message>
    <message name="doSthResponse">
        <part name="parameters" element="tns:doSthResponse"/>
    </message>
    <portType name="ServerDemo">
        <operation name="sayHi">
            <input wsam:Action="http://demo/ServerDemo/sayHiRequest" message="tns:sayHi"/>
            <output wsam:Action="http://demo/ServerDemo/sayHiResponse" message="tns:sayHiResponse"/>
        </operation>
        <operation name="doSth">
            <input wsam:Action="http://demo/ServerDemo/doSthRequest" message="tns:doSth"/>
            <output wsam:Action="http://demo/ServerDemo/doSthResponse" message="tns:doSthResponse"/>
        </operation>
    </portType>
    <binding name="ServerDemoPortBinding" type="tns:ServerDemo">
        <soap:binding transport="http://schemas.xmlsoap.org/soap/http" style="document"/>
        <operation name="sayHi">
            <soap:operation soapAction=""/>
            <input>
                <soap:body use="literal"/>
            </input>
            <output>
                <soap:body use="literal"/>
            </output>
        </operation>
        <operation name="doSth">
            <soap:operation soapAction=""/>
            <input>
                <soap:body use="literal"/>
            </input>
            <output>
                <soap:body use="literal"/>
            </output>
        </operation>
    </binding>
    <service name="ServerDemoService">
        <port name="ServerDemoPort" binding="tns:ServerDemoPortBinding">
            <soap:address location="http://localhost:8080/SOAPServer/firstSoap"/>
        </port>
    </service>
</definitions>
```
获取到wsdl文档之后，我们使用JDK自带的wsimport实用程序来生成客户端代码。（与之相反的是，wsgen实用程序可以通过服务端代码来生成wsdl文件，wsgen和wsimport命令详解可参考[[wsgen和wsimport命令讲解](http://blog.csdn.net/yuguiyang1990/article/details/9003383)](http://blog.csdn.net/yuguiyang1990/article/details/9003383)）
在本例中使用的是windows系统，打开CMD，输入
```shell
wsimport -s D:\work_eclipse\SOAPClient\src\main\java -p my.client http://localhost:8080/SOAPServer/firstSoap?wsdl
```
即在D盘work_eclipse\SOAPClient\src\main\java目录下的my.client目录中生成了客户端代码（工程SOAPClient是新建的用来调用SOAPServer的工程），目录如图所示：
![根据wsdl文件生成的客户端代码（client包下）](http://upload-images.jianshu.io/upload_images/3727888-4d2ff98e7b88713a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在服务端代码中，ServerDemo服务类，对应客户端中生成的ServerDemo接口和ServerDemoService类；服务类中的方法sayHi()对应客户端代码中的SayHi和SayHiResponse实体类；服务类中的方法doSth()对应客户端代码中的DoSth和DoSthResponse实体类。客户端中还生成了ObjectFactory类和package-info类。此处先不关心生成的类的结构，先尝试进行调用（当然，此时要保证SOAP服务处于已发布的状态）。

在SOAPclient的call目录下创建调用类：
```java
package call;

import my.client.ServerDemo;
import my.client.ServerDemoService;

public class ClientDemo {
	public static void main(String[] args) {
		ServerDemoService server = new ServerDemoService();
		ServerDemo serverDemo = server.getServerDemoPort();
		System.out.println(serverDemo.sayHi("erver body"));
		System.out.println(serverDemo.doSth());
	}
}
```
执行main方法之后，会输出“Hi: erver body,this is SOAPServer”与“do some thing”两句话，即说明调用成功，实现了远程调用。

#### 4\. SOAP报文和WSDL文档简要解析。
为了直观，我们先只分析调用sayHello.sayHi()时候的报文，在HelloClient 中注释掉sayHello.doSth()，然后发起请求，调用sayHi()方法。通过tcpTrace截获的SOAP请求报文如下：
```xml
GET /SOAPServer/firstSoap?wsdl HTTP/1.1
User-Agent: Java/1.8.0_60
Host: localhost:8080
Accept: text/html, image/gif, image/jpeg, *; q=.2, */*; q=.2
Connection: keep-alive

POST /SOAPServer/firstSoap HTTP/1.1
Accept: text/xml, multipart/related
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://demo/ServerDemo/sayHiRequest"
User-Agent: JAX-WS RI 2.2.9-b130926.1035 svn-revision#5f6196f2b90e9460065a4c2f4e30e065b245e51e
Host: localhost:8080
Connection: keep-alive
Content-Length: 187


<?xml version="1.0" ?>
<S:Envelope  xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
    <S:Body>
        <ns2:sayHi xmlns:ns2="http://demo/">
            <arg0>erver body</arg0>
        </ns2:sayHi>
    </S:Body>
</S:Envelope>
```
通过tcpTrace截获的SOAP返回报文如下：
```xml
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Content-Type: text/xml;charset=utf-8
Transfer-Encoding: chunked
Date: Sat, 29 Jul 2017 17:07:52 GMT

<?xml version='1.0' encoding='UTF-8'?>
<!-- Published by JAX-WS RI (http://jax-ws.java.net). RI's version is JAX-WS RI 2.2.10 svn-revision#919b322c92f13ad085a933e8dd6dd35d4947364b. -->
<!-- Generated by JAX-WS RI (http://jax-ws.java.net). RI's version is JAX-WS RI 2.2.10 svn-revision#919b322c92f13ad085a933e8dd6dd35d4947364b. -->
<definitions
    xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd"
    xmlns:wsp="http://www.w3.org/ns/ws-policy"
    xmlns:wsp1_2="http://schemas.xmlsoap.org/ws/2004/09/policy"
    xmlns:wsam="http://www.w3.org/2007/05/addressing/metadata"
    xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
    xmlns:tns="http://demo/"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns="http://schemas.xmlsoap.org/wsdl/" targetNamespace="http://demo/" name="ServerDemoService">
    <types>
        <xsd:schema>
            <xsd:import namespace="http://demo/" schemaLocation="http://localhost:8080/SOAPServer/firstSoap?xsd=1"/>
        </xsd:schema>
    </types>
    <message name="doSth">
        <part name="parameters" element="tns:doSth"/>
    </message>
    <message name="doSthResponse">
        <part name="parameters" element="tns:doSthResponse"/>
    </message>
    <message name="sayHi">
        <part name="parameters" element="tns:sayHi"/>
    </message>
    <message name="sayHiResponse">
        <part name="parameters" element="tns:sayHiResponse"/>
    </message>
    <portType name="ServerDemo">
        <operation name="doSth">
            <input wsam:Action="http://demo/ServerDemo/doSthRequest" message="tns:doSth"/>
            <output wsam:Action="http://demo/ServerDemo/doSthResponse" message="tns:doSthResponse"/>
        </operation>
        <operation name="sayHi">
            <input wsam:Action="http://demo/ServerDemo/sayHiRequest" message="tns:sayHi"/>
            <output wsam:Action="http://demo/ServerDemo/sayHiResponse" message="tns:sayHiResponse"/>
        </operation>
    </portType>
    <binding name="ServerDemoPortBinding" type="tns:ServerDemo">
        <soap:binding transport="http://schemas.xmlsoap.org/soap/http" style="document"/>
        <operation name="doSth">
            <soap:operation soapAction=""/>
            <input>
                <soap:body use="literal"/>
            </input>
            <output>
                <soap:body use="literal"/>
            </output>
        </operation>
        <operation name="sayHi">
            <soap:operation soapAction=""/>
            <input>
                <soap:body use="literal"/>
            </input>
            <output>
                <soap:body use="literal"/>
            </output>
        </operation>
    </binding>
    <service name="ServerDemoService">
        <port name="ServerDemoPort" binding="tns:ServerDemoPortBinding">
            <soap:address location="http://localhost:8080/SOAPServer/firstSoap"/>
        </port>
    </service>
</definitions>

HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Content-Type: text/xml;charset=utf-8
Transfer-Encoding: chunked
Date: Sat, 29 Jul 2017 17:07:52 GMT


<?xml version='1.0' encoding='UTF-8'?>
<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
    <S:Body>
        <ns2:sayHiResponse  xmlns:ns2="http://demo/">
            <return>Hi: erver body,this is SOAPServer</return>
        </ns2:sayHiResponse>
    </S:Body>
</S:Envelope>
```
从报文格式可以看出，明显的HTTP+XML格式，对于SOAP1.1和SOAP1.2相比，在HTTP格式的请求头里面会有些区别，主要是命名空间的区别。可以粗略参考[SOAP1.1和SOAP1.2报文差异](http://blog.sina.com.cn/s/blog_5f044a4d0101gzli.html)。从报文里面可以看出，虽然ClientDemo只调用sayHi()方法，但是请求有两段，第一个get请求，是请求返回的wsdl文档；第二个请求是post请求，才是真正调用sayHi()方法，并返回了结果。实际上，发起get请求去获得对应的wsdl文档这个动作，是在客户端创建ServerDemoService这个对象的时候发起的，也就是下面这行代码：
```java   
ServerDemoService server = new ServerDemoService(); 
 ```
显然，在soap中，wsdl文档至关重要，包含了服务端的所有调用信息。从结构上看，最顶层是definitions根节点，根节点下面分为以下几部分：
#####  types部分
从wsdl文档中可以看出，types节点指向了一个地址：“http://localhost:8080/SOAPServer/firstSoap?xsd=1”，而这个地址是一个schema文件（即一个描述XML格式的文件），该文件内容如下：
```xml
<xs:schema version="1.0" targetNamespace="http://demo/">
    <xs:element name="doSth" type="tns:doSth"/>
    <xs:element name="doSthResponse" type="tns:doSthResponse"/>
    <xs:element name="sayHi" type="tns:sayHi"/>
    <xs:element name="sayHiResponse" type="tns:sayHiResponse"/>
    <xs:complexType name="doSth">
        <xs:sequence/>
    </xs:complexType>
    <xs:complexType name="doSthResponse">
        <xs:sequence>
            <xs:element name="return" type="xs:string" minOccurs="0"/>
        </xs:sequence>
    </xs:complexType>
    <xs:complexType name="sayHi">
        <xs:sequence>
            <xs:element name="arg0" type="xs:string" minOccurs="0"/>
        </xs:sequence>
    </xs:complexType>
    <xs:complexType name="sayHiResponse">
        <xs:sequence>
            <xs:element name="return" type="xs:string" minOccurs="0"/>
        </xs:sequence>
    </xs:complexType>
</xs:schema>
```
文件里规定了元素名称、元素类型。以sayHi为例，
```xml
<xs:element name="sayHi" type="tns:sayHi"/>
```
说明了有一个元素名为sayHi，这个元素的类型是“tns:sayHi”，而“tns:sayHi”指向下面这段：
```xml
<xs:complexType name="sayHi">
    <xs:sequence>
        <xs:element name="arg0" type="xs:string" minOccurs="0"/>
    </xs:sequence>
</xs:complexType>
```
说明sayHi是一个复杂类型，而这个复杂类型再包含了一个元素，元素名为arg0，arg0的类型是String类型（minOccurs表示这个元素“最小出现次数”）。
结合请求报文中的Body节点：
```xml
<?xml version="1.0" ?>
<S:Envelope  xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
    <S:Body>
        <ns2:sayHi xmlns:ns2="http://demo/">
            <arg0>erver body</arg0>
        </ns2:sayHi>
    </S:Body>
</S:Envelope>
```
可以看出，上面的type就是描述了请求报文和返回报文的节点名称、节点内容的类型。
##### portType和message
portType节点中主要包含若干个operation节点（每一个operation节点与服务方的方法对应）。operation节点中包括input和output，input在output之前，即是请求/响应模式。input和output都有一个message属性，这个message属性指向了相应的message节点。而这个message节点中又包含一个part，part的element属性指定了这个operation节点的输入值的参数名以及类型。简而言之，portType和message分别指定了这个服务拥有的方法，以及每个方法需要的参数和返回值的类型以及名称。
```xml
<portType name="ServerDemo">
        <operation name="doSth">
            <input wsam:Action="http://demo/ServerDemo/doSthRequest" message="tns:doSth"/>
            <output wsam:Action="http://demo/ServerDemo/doSthResponse" message="tns:doSthResponse"/>
        </operation>
        <operation name="sayHi">
            <input wsam:Action="http://demo/ServerDemo/sayHiRequest" message="tns:sayHi"/>
            <output wsam:Action="http://demo/ServerDemo/sayHiResponse" message="tns:sayHiResponse"/>
        </operation>
    </portType>
```

```xml
    <message name="doSth">
        <part name="parameters" element="tns:doSth"/>
    </message>
    <message name="doSthResponse">
        <part name="parameters" element="tns:doSthResponse"/>
    </message>
```
##### binding
transport的值 “http://schemas.xmlsoap.org/soap/http” 可以概括为基于HTTP的SOAP，这个值还可以是其他的值，如SMTP、TCP。style为document，指定这个服务的style为document（意味着消息格式是XML格式或者等价的文档格式）。style还有一个选项是“rpc”，这个名称容易引起误解，这个值意味着消息本身不是类型化的（没有type部分），而use="literal"是WSDL中定义的服务的类型遵循WSDL的模式，决定了数据类型被如何编码和解码。
```xml
    <binding name="ServerDemoPortBinding" type="tns:ServerDemo">
        <soap:binding transport="http://schemas.xmlsoap.org/soap/http" style="document"/>
        <operation name="doSth">
            <soap:operation soapAction=""/>
            <input>
                <soap:body use="literal"/>
            </input>
            <output>
                <soap:body use="literal"/>
            </output>
        </operation>
        <operation name="sayHi">
            <soap:operation soapAction=""/>
            <input>
                <soap:body use="literal"/>
            </input>
            <output>
                <soap:body use="literal"/>
            </output>
        </operation>
    </binding>
```
##### service
以上的部分更多的是在指定服务的名称、类型等等规范。而service提供了服务的相关实现细节。service中包含port子元素（数量由binding部分的数量，因为明显看出port是指向相应的binding元素的），最重要的还是address子元素，指定了一个location，即发布服务的地址，通过解析这个地址进行调用。
```xml
    <service name="ServerDemoService">
        <port name="ServerDemoPort" binding="tns:ServerDemoPortBinding">
            <soap:address location="http://localhost:8080/SOAPServer/firstSoap"/>
        </port>
    </service>
```
通过这个WSDL文档，描述了完整的服务信息。这些信息是可以在服务类里面通过注解进行配置的。例如@  SOAPBinding可以将binding部分的style属性由“document”更改为“rpc”。详细的配置本文就不深入了，因为现实中已经很少会使用这种JAX-WS实现，JAX-WS实现已经被其他的框架所扩展。下一篇文章将简单介绍基于SOAP的webservice的Axis2框架实现。
















