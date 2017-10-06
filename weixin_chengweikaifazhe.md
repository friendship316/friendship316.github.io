---
title: 微信java开发系列 一、认证成为开发者
date: 2016-12-07
---
- 声明：本文的操作过程参考了很多网络资源，但由于记录时已失去链接，因此无法贴出。
- 因为业务需求，学习微信开发，网上很多资源对我来说可能不够细节，于是我把我做过的和想到的有用的东西记录于此，希望能对刚开始微信开发的朋友们有参考作用。

---
- 微信双向认证的过程就是两台服务器打招呼的过程，就好像我们接下来要一起做事，那么我们需要首先打个招呼，彼此知道对方在哪里一样。

## 一、准备工作
  1. **需要一个自己的服务器资源和域名。**我是搭建“科学上网”环境的时候自己买的一个VPS，并在上面搭建的nginx和tomcat，相关教程网络上资源很多。如果个人没有服务器资源的话，可以使用像BAE这类的应用引擎，相关资源请百度或者谷歌一下，本文默认您拥有自己的服务器资源并且已经搭建好web服务环境。 
  2. **登录微信公众平台并注册账号，申请一个订阅号或者服务号。**个人只能选择订阅号，订阅号和服务号的区别是服务号拥有更多的接口调用权限。如果你拥有申请服务号的资格，那么直接申请服务号即可。如果你只是自己学习开发，那么就申请订阅号，我们可以在自己的订阅号里面申请一个测试号，测试号拥有全部的接口调用权限，完全可以满足开发者学习。本文默认申请的是订阅号。

##  二、认证成为开发者
####  (一)、了解相关参数以及申请测试号
*  **熟悉微信开发配置参数**。注册并申请订阅号之后，进入个人页面，选择开发→基本配置，可以看到相关配置。
 1. 开发者ID（包括AppID和AppSecret）。这里简单说一下这两个参数，调用微信接口的时候需要使用这两个参数获取一个access_token，有了这个access_token，相当于我们才拥有了使用微信接口的权限。我们每次调用微信接口都需要传过去我们的access_token以供微信服务器验证。
 2. 除了开发者ID以外，我们还能看见服务器配置选项，点击修改配置，会进入填写自己的服务器信息界面。有三个参数：
    + URL——你的服务器上接收微信消息的Servlet的路径（到方法）。相当于Struts2的Action路径（到方法），相当于SpringMVC的Controller的路径（到方法）。比如我使用的SpringMVC，URL是http://www.xxxx.com/web工程名/weixin/test.do ，这里还有一点是URL可以以http和https开头，分别支持80端口和443端口。如果使用的是http开头，那么你的服务器上面的web服务端口应该是80。如果以https开头，那么你的服务器上面的web服务端口应该是443。
     + Token——用于验证的参数。由用户自定义。
     +   EncodingAESKey——用于加解密的字符串，如果我们选择EncodingAESKey参数下方的消息加解密方式为安全模式，那么我们接收到的微信发送过来的消息是经过加密的，同时我们发送给微信服务器的消息也应该加密，加解密的示例在[微信开放平台的资源中心](https://open.weixin.qq.com/cgi-bin/showdocument?action=dir_list&t=resource/res_list&verify=1&id=open1419318479&token=&lang=zh_CN)有源代码可供下载（包括c++, php, java, python, c# 5种语言的示例代码）。
* **申请测试号**，为了拥有全部接口调用权限，我们可以申请一个测试号。在微信公众平台的个人页面选择开发下面的开发者工具，选择公众平台测试账号，进入申请测试号的管理页面之后，我们可以看到同样有appID和appsecret，同时也有接口配置信息，这些参数和上文提到的同名参数作用是一致的，即测试号也是一个公众号，只不过测试号有最多允许100个用户关注，以及名字无法自定义等限制，而且没有消息加解密功能。如下图：
![测试号管理界面](http://upload-images.jianshu.io/upload_images/3727888-61634847ea907b3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####  (二)、双向认证
* 啰嗦了半天，终于可以步入正题了。我们需要完成双向认证才能进行开发，此处相当于微信服务器需要发送一条消息在我们自己的服务器上，我们自己的服务器接收消息之后，首先确认这条消息是否是微信发送（这相当于我们自己的服务器验证微信，当然其实如果仅仅是想要微信服务器认证我们——单向认证，也可以不需要这一步），如果是，我们再原样返回微信发送的echostr参数（随机字符串），这样微信服务器接收消息之后，就认证了我这台服务器。  双向认证完成，就可以开始开发了。 
* 为此，我们需要创建一个web工程写一个Servlet服务接收微信服务器的请求，如果使用框架，即是SpringMVC的Controller，Struts2的Action，我使用的是SpringMVC，为了便于熟悉流程，将所有用到的东西都放在一个类中。没有进行分解，封装工具类等等，本文使用的是JDK1.8，注释详细，基本都能看懂。

代码会说话：
```
package my.controller;

import java.io.IOException;
import java.io.PrintWriter;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Arrays;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.apache.log4j.Logger;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
@Controller
@RequestMapping("/weixin")
public class GetWeixinController {
	
	private static Logger logger = Logger.getLogger(GetWeixinController.class);
	
	@RequestMapping("/test")
	public void connect(HttpServletRequest request,HttpServletResponse response){
		logger.info("接入成功");
		String token = "lifasen";//在开发者的测试号管理页面填写的token
		
		//获取请求中的参数，由于测试号没有加解密功能，所以此处并不需要解密，直接获取
		String timestamp = request.getParameter("timestamp");
		String nonce = request.getParameter("nonce");
		String weixinSignature = request.getParameter("signature");
		String echostr = request.getParameter("echostr");
		logger.info("参数：token:"+token+",timestamp:"+timestamp+",nonce:"+nonce+",signature:"+weixinSignature);
		
		//对参数进行字典序排序
		String[] para = {token,timestamp,nonce};
		Arrays.sort(para);
		
		//排序之后转换成字符串
		String paraString = "";
		for(int i=0;i<para.length;i++){
			paraString += para[i];
		}
		logger.info("排序结果："+paraString);
		
		//sha1加密工具(java.security.MessageDigest)
		MessageDigest messageDigest = null;	
		try {
			//获取加密工具对象
			messageDigest = MessageDigest.getInstance("SHA-1");
			//加密并转换为16进制字符串，byteToStr方法详细内容见下方
			String mySignature = byteToStr(messageDigest.digest(paraString.getBytes()));
			if(mySignature.equals(weixinSignature.toUpperCase())){
				logger.info("==========验证成功============");
				//原样返回微信服务器发送的echostr参数，同样，不需要加密
				PrintWriter out = response.getWriter();
				out.print(echostr);
				logger.info("==========写出返回值成功============");
			}
		} catch (IOException e) {
			logger.error("获取输出流失败", e);
		} catch (NoSuchAlgorithmException e) {
			logger.error("获取加密工具对象失败",e);
		}
	}
	/**
	 * 将字节转换为十六进制字符串 
	 * @param mByte
	 * @return
	 */
	private static String byteToHexStr(byte mByte) {  
        char[] Digit = { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F' };  
        char[] tempArr = new char[2];  
        tempArr[0] = Digit[(mByte >>> 4) & 0X0F];  
        tempArr[1] = Digit[mByte & 0X0F];  
        String s = new String(tempArr);  
        return s;  

    }  
	/**
	 * 将字节数组转换为十六进制字符串
	 * @param byteArray
	 * @return
	 */
	private static String byteToStr(byte[] byteArray) {  

	        String strDigest = "";  
	        for (int i = 0; i < byteArray.length; i++) {  
	            strDigest += byteToHexStr(byteArray[i]);  
	        }  
	        return strDigest;  

	    }  
	
}

```
* 至此，将工程打包，部署，在我们的服务器上启动tomcat，然后在测试号管理界面填写并点击提交，可以看见测试号管理界面提示配置成功，此时我们的服务器和微信服务器的双向认证成功，可以开始开发了。

![测试号管理界面](http://upload-images.jianshu.io/upload_images/3727888-3beee7a6e7b2d8d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![tomcat后台](http://upload-images.jianshu.io/upload_images/3727888-3735133eb7e31b18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 如果配置失败，请详细查看tomcat错误日志，排查错误，比如访问路径不对等等。