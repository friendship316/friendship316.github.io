---
title: maven摸索记录
date: 2016-03-06 17:30:32
tags:
---
### 动机：
	1、新手上路，想把公司的maven项目导入到家里另一台不能连公司私服的电脑上。
	2、想了解公司使用的nexus私服，于是尝试了搭建过程。
### 所用环境：
	本文所用eclipse为mars1版本，并配置了最新maven 3.3.9版本插件，Win10系统。
### 分析：
	将公司的maven项目导入家里电脑上有两种方式：
	1、将公司的项目打成war包然后导入即可，方式简单，但是每次都要copy大量的jar包。
	2、在家里电脑上搭建nexus私服，首次部署需要将公司电脑里面的maven的仓库repository文件夹连同项目一起copy。
### 方式1 打war包步骤：
	(1)、在eclipse项目上右键选择run as，选择Maven build...
![](http://7xrt1z.com1.z0.glb.clouddn.com/mavenwarStep1.png)
	(2)、输入clean package命令，点击run。
![](http://7xrt1z.com1.z0.glb.clouddn.com/mavenwarStep2.jpg)	
	(3)、在eclipse的工作区间中，项目文件夹下的target文件夹下会生成-项目名.war文件，复制这个文件。
	(4)、在家里eclipse上导入war包，file→import→war file→Browse选择war包所在位置即可。
### 方式2 nexus私服搭建步骤：
	(1)首先从http://www.sonatype.org/nexus/下载最新版本的nexus，国外的网站可能访问速度会比较慢。
	分别有Bundle包和war包两种格式，Bundle包自带了Jetty容器，因此用户不需要额外的web容器就可以
	直接启动nexus，war包格式则需要部署到额外的web容器，该war包支持主流的web容器如Tomcat、Jetty、
	Glassfish、Resin。本文下载的是最新nexus-2.12.0-01版本的bundle包(windows系统)。下载之后解
	压文件得到两个子目录：
![](http://7xrt1z.com1.z0.glb.clouddn.com/nexusstep1.png)
	**nexus-2.12.0-01**目录包含了nexus运行所需要的文件。**sonatype-work**目录包含nexus生成的配置文件，该目录不是必须的，nexus在运行的时候会动态创建该目录，但是它的内容包含了用户自己的配置和仓库内容。
	(2)在jsw文件夹下有很多文件夹，存放着各个系统的批处理文件，我这里选择Windows64位，进入后依次运行**console-nexus.bat**→**install-nexus.bat**→**start-nexus.bat**。
![](http://7xrt1z.com1.z0.glb.clouddn.com/nexusstep2.png)
![](http://7xrt1z.com1.z0.glb.clouddn.com/nexusstep3.png)
	(3)完成后用浏览器访问  http://127.0.0.1:8081/nexus  有如下界面即成功，点击右上角登录，用户名为admin，密码为admin123。ps：8081是默认的端口号，进入nexus-2.12.0-01\conf\打开nexus.properties文件，修改application-port属性值即可修改端口号。
![](http://7xrt1z.com1.z0.glb.clouddn.com/nexusstep4.png)
	(4)登录后，点击左侧的Repositories可以浏览现有仓库，这个列表包含了所有类型的nexus仓库，仓库有四种类型：group(仓库组)、host(宿主)、proxy(代理)和virtual(虚拟)。每个仓库的格式为maven2或者maven1。属性Policy表示该仓库为发布(release)版本还是快照(snapshot)版本，最后两列为仓库的状态和路径。
	各个仓库的简要介绍如下：
		  Public Repositories:  仓库组
		  3rd party: 无法从公共仓库获得的第三方发布版本的构件仓库
		  Apache Snapshots: 用了代理ApacheMaven仓库快照版本的构件仓库
		  Central: 用来代理maven中央仓库中发布版本构件的仓库
		  Central M1 shadow: 用于提供中央仓库中M1格式的发布版本的构件镜像仓库
		  Codehaus Snapshots: 用来代理CodehausMaven 仓库的快照版本构件的仓库
		  Releases: 用来部署管理内部的发布版本构件的宿主类型仓库
		  Snapshots:用来部署管理内部的快照版本构件的宿主类型仓库
	各个类型的仓库之间的联系如下图所示(本图来自《maven实战》)：
![](http://7xrt1z.com1.z0.glb.clouddn.com/nexusrelation.jpg)
	maven可以直接从宿主仓库下载构件，也可以从代理仓库下载构件，而代理仓库会间接从远程仓库下载并缓存构件。为了方便，maven可以从仓库组下载构件，而仓库组没有实际内容，它会转向其包含的宿主仓库或代理仓库获得实际构件的内容。

	点击此页菜单栏上的Add按钮会出现下图所示，可以分别添加四种仓库。
![](http://7xrt1z.com1.z0.glb.clouddn.com/nexusadd.png)
	例如选择Hosted Repository可以添加宿主仓库，仓库列表下方会出现如下配置：
![](http://7xrt1z.com1.z0.glb.clouddn.com/nexushost.png)
	点击save后会在仓库列表中看到刚刚新增的仓库。由于是个人使用，我只创建了一个宿主仓库。创建其他类型的仓库详见《maven实战》第9章。
	(5)配置maven从nexus下载构件。
	此处有两种方式，第一种是在项目的pom中配置nexus仓库，但是只对maven项目有效。第二种是profile机制配置到setting.xml中，可以让本机所有的maven项目都使用自己的maven私服。
	未完待续。。。

