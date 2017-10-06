---
title: 微信Java开发系列 二、接收并回复文本消息
date: 2016-12-24
---
- 声明：本文的操作过程参考了很多网络资源，但由于记录时已记不清参考的哪些资源，因此无法贴出。
- 因为业务需求，学习微信开发，网上很多资源对我来说可能不够细节，于是我把我做过的和想到的有用的东西记录于此，可能有些内容比较啰嗦，但希望能对刚开始微信开发的朋友们有参考作用，感谢每一位点击的朋友，我们一起进步。

---
## 功能简介
* 被动回复消息功能，即用户向公众号发送消息，公众号进行自动回复的功能。实际的过程是用户发送消息到微信，微信服务器接收之后，由于我们配置并认证了自己的服务器，于是微信会发送一个请求到我们配置的服务器上，同时会以XML格式传递一系列参数，开发者通过解析XML来获取用户的消息内容。然后需要以[微信开发者文档](https://mp.weixin.qq.com/wiki)规定的格式构造并发送回复内容到微信服务器。
* 这里需要说明的是，用户给公众号发送的消息会有很多种类型，目前微信公众号支持的消息类型有：文本消息、图片消息、语音消息、视频消息、小视频消息、地理位置消息、链接消息。对于不同的消息类型，微信服务器发送给我们的对应的XML数据包的格式会略有不同。
* 同样，我们被动回复给用户也可以回复不同类型的消息，针对不同的消息需要构造的XML数据包的格式也有所不同。
* 更多详情请见[微信开发者文档](https://mp.weixin.qq.com/wiki)→消息管理。

## 开始开发
* 我的上一篇文章[微信java开发系列 一、认证成为开发者](http://www.jianshu.com/p/7deaddd39a92)中是写了一个servlet来接收微信服务器发送给我们的消息，并将其访问路径配置为微信公众号中的服务器配置的URL参数。在本文的接收消息的时候，我们依旧需要这个URL参数。认证成为开发者之后，在以后的接收消息的时候就不再需要认证了，我们可以选择将那一个servlet的路径注释掉，或者干脆将其中的代码挪至其他地方保存。
* 这里我们新建一个servlet，其访问路径就是我们在微信公众号中配置的URL。首先以接收文字消息以及回复文字消息为例。其XML数据包格式如下：
![接收到的文本消息的XML数据包格式](http://upload-images.jianshu.io/upload_images/3727888-e8c4c013bdf9db9d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. 我们首先需要做的是解析XML，为了流程清晰，这里简单的封装一下，把解析XML数据包封装成一个方法（工具类WeixinUtils）。
```java
/**
 * 解析微信请求并读取XML
 * @param request
 * @return
 * @throws IOException
 * @throws DocumentException
 */
public static Map<String,String> readWeixinXml(HttpServletRequest request) throws IOException, DocumentException{
	Map<String,String> map = new HashMap<String,String>();
	//获取输入流
	InputStream input = request.getInputStream();
	//使用dom4j的SAXReader读取（org.dom4j.io.SAXReader;）
	SAXReader sax = new SAXReader();
	Document doc = sax.read(input);
	//获取XML数据包根元素
	Element root = doc.getRootElement();
	//得到根元素的所有子节点
	@SuppressWarnings("unchecked")
	List<Element> elementList = root.elements();
	//遍历所有节点并将其放进map
	for(Element e : elementList){
		map.put(e.getName(), e.getText());
	}
	//释放资源
	input.close();
	input = null;
	return map;
	
}
```
2. 获取内容并回复文本内容。
```java
package my.controller;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.Map;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.apache.log4j.Logger;
import org.dom4j.DocumentException;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import my.Util.WeixinUtils;
@Controller
@RequestMapping("/weixin")
public class WeixinMessage {
	private Logger log = Logger.getLogger(WeixinMessage.class);
	@RequestMapping("/test")
	public void replyTextMessage(HttpServletRequest request,HttpServletResponse response){
		Map<String,String> map = null;
		//从工具类中获取XML解析之后的map
		try {
			map =  WeixinUtils.readWeixinXml(request);
		} catch (IOException e) {
			log.error("获取输入流失败", e);
		} catch (DocumentException e) {
			log.error("读取XML失败", e);
		}
		//获取发送方账号
		String fromUserName = map.get("FromUserName");
		//接收方账号(开发者微信号)
		String toUserName = map.get("ToUserName");
		//消息类型
		String msgType = map.get("MsgType");
		//文本内容
		String content = map.get("Content");
		log.info("发送方账号:"+fromUserName+",接收方账号(开发者微信号):"+toUserName+",消息类型:"+msgType+",文本内容:"+content);
		//回复消息
		if(msgType.equals("text")){
			//根据开发文档要求构造XML字符串，本文为了让流程更加清晰，直接拼接
			//这里在开发的时候可以优化，将回复的文本内容构造成一个java类
			//然后使用XStream(com.thoughtworks.xstream.XStream)将java类转换成XML字符串，后面将会使用这个方法。
			//而且由于参数中没有任何特殊字符，为简单起见，没有添加<![CDATA[xxxxx]]>
			String replyMsg = "<xml>"+
								"<ToUserName>"+fromUserName+"</ToUserName>"+
								"<FromUserName>"+toUserName+"</FromUserName>"+
								"<CreateTime>"+System.currentTimeMillis()/1000+"</CreateTime>"+
								"<MsgType>"+msgType+"</MsgType>"+
								"<Content>"+content+"</Content>"+
							 "</xml>";
			//响应消息
			log.info("响应消息："+replyMsg);
			PrintWriter out = null;
			try {
				//设置回复内容编码方式为UTF-8，防止乱码
				response.setCharacterEncoding("UTF-8");
				out = response.getWriter();
				//我们这里将用户发送的消息原样返回
				out.print(replyMsg);
				log.info("==============响应成功==================");
			} catch (IOException e) {
				log.error("获取输出流失败",e);
			}finally {
				if(out != null){
					out.close();
					out = null;
				}
			}
		}
	}
}
```  
3. 打包，部署，向自己的测试号发送消息，可看到如下：

![接收并回复文本消息](http://upload-images.jianshu.io/upload_images/3727888-e153db10dccb31fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![tomcat后台记录](http://upload-images.jianshu.io/upload_images/3727888-99c607195c17fd29.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


*  到此，接收消息回复消息的功能就完成了，本文只是记录大概的后台流程，并没有涉及过多的细节，我们如果要回复其他类型的消息，只需要构造其他类型的XML字符串即可，需要注意的是**“图片消息”和“图文消息”两者对应的XML格式不同**。另外，本文没有排除重复消息，所以没有使用MsgId这个参数，如果两次消息的MsgId一样，那么就是重复发送的同一条消息。