---
title: 生成自定义Docker镜像
date: 2019-03-14 00:30:47
---

- 本文记录生成自己的Docker镜像的两个例子。第一个是在一个SpringBoot工程基础上生成镜像，第二个是在普通的基于Tomcat的Web工程的基础上生成镜像。

---

### Docker常用命令

列出所有镜像（加上参数`-a` 可以包括中间层镜像，还有`-q`和`--format` 等实用参数）
`docker image ls`

查看镜像、容器、数据卷所占用的空间
`docker system df`

删除镜像
`docker image rm [选项] <镜像1> [<镜像2> ...]`

启动镜像
`docker run [OPTIONS] <镜像>`
`[OPTIONS]`常用参数：
`-d`: 后台运行容器，并返回容器ID；
`-i`: 以交互模式运行容器，通常与 -t 同时使用；
`-t`: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
`-p`: 端口映射，格式为：主机(宿主)端口:容器端口；
`--name "xxx"`: 为容器指定一个名称；

以交互式终端方式进入容器
`docker exec -it <容器名>`

使用 Dockerfile 创建镜像，这个后面必须跟参数，且须有一个`.`
`docker build`

### 生成SpringBoot的Web工程的镜像

#### 创建SpringBoot的工程
首先创建一个简单的SpringBoot工程，工程很简单，功能是访问localhost:8080从而跳转到指定页面。工程中只有一个TestController，这个TestController的默认路径指向index.html页面，这个index.html在resources目录下的templates.home中。static.home可以忽略，其中存的只是index.html的样式文件。

工程目录结构如下：

![SpringBoot工程目录结构](https://upload-images.jianshu.io/upload_images/3727888-bc629fd3d820a9a1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

TestController的内容，没有任何逻辑，只是跳转到index.html页面：

```
package com.example.learndocker.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("/")
public class TestController {

    @RequestMapping(value = "")
    public String home() {
        return "home/index";
    }
}
```

为了返回页面，需要引用模板引擎，pom.xml如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>learn-docker</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>learn-docker</name>
    <description>sth about docker</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>2.1.3.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-thymeleaf -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
            <version>2.1.3.RELEASE</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

```

执行LearnDockerApplication的main方法，在浏览器访问localhost:8080，可以看到如下页面:

>![SpringBoot启动后访问localhost:8080](https://upload-images.jianshu.io/upload_images/3727888-3dd2b47603861297.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 生成Docker镜像

接下来生成docker镜像。首先将SpringBoot打成jar包，然后在任意位置创建Dockerfile文件，我是在learn-docker的工程目录下创建的，为了方便，我直接在IDEA中编辑了。

添加了Dockerfile文件，并且打包工程之后，工程目录结构如下：

![Dockerfile所在目录](https://upload-images.jianshu.io/upload_images/3727888-3541259b7d87f2ab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Dockerfile所在目录](https://upload-images.jianshu.io/upload_images/3727888-778439b7d933fa69.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Dockerfile内容如下：
```text
#docker仓库的JDK镜像
FROM java:8
#挂载点，即容器中/tmp(这个目录是springboot内嵌的Tomcat默认使用目录)的改动会同步到主机的指定目录
VOLUME /tmp
#将自己的jar包加入生成的镜像中（在JDK这一层的基础上加上我们的jar构建新的镜像）
ADD target/learn-docker-0.0.1-SNAPSHOT.jar learn-docker.jar
#容器启动完执行
ENTRYPOINT ["java","-jar","learn-docker.jar"]
```

分析：运行springboot的web工程，需要操作系统+JDK/JRE，所以需要以操作系统镜像作为基础，然后加入JDK/JRE，再加上自己的jar，生成自定义镜像。

但是因为Docker镜像是分层构建，所以官方的java:8镜像中已经包含了操作系统的镜像这一层，所以此处直接引用此基础镜像即可。然后加入我们自己的jar，在基础镜像的基础上生成新的镜像。并设置当启动容器时，启动springboot的jar包即可。

当然，如果不想使用现有的Java镜像，也可以下载好JDK/JRE，并以现有的操作系统的镜像为基础（有很多优化过的体积很小的操作系统基础镜像，比如alpine），加入JDK/JRE，同时加入我们自己的jar，配置好Java环境变量，生成新的镜像，这样会使得新镜像的层数少一层，原则上是层数越少越好。

执行命令`docker build -t learndocker:v1`构建镜像，注意命令后面有一个`.`不能少，构建过程：

![镜像构建过程](https://upload-images.jianshu.io/upload_images/3727888-ce616ae3e7a737e1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

构建完毕之后：

![构建镜像完毕](https://upload-images.jianshu.io/upload_images/3727888-b0130bb7aab46304.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时`docker images`查看镜像，可以看到刚构建完的自己命名的learndocker:v1，当然同时也下载了java:8的镜像（这里可以看出分层构建镜像的思想）。kafka镜像是本人之前下载的。

![查看镜像.jpg](https://upload-images.jianshu.io/upload_images/3727888-bdb0626ecb0067a9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时可以正常启动容器并将宿主的8080端口与镜像的8080端口映射，执行命令`docker run -p 8080:8080 <镜像id>`，此时我并没有让容器在后台启动，所以当我退出终端时，容器会停止。

![启动容器](https://upload-images.jianshu.io/upload_images/3727888-78dbe21ab1b0f6b8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

保持终端不关闭，此时访问浏览器localhost:8080，可以看到与之前同样的页面，整个定制镜像并启动的过程成功。

为了理解挂载点的意思，在启动时使用`docker run -v /Users/shakeli/idea_workspace/learn-docker/test:/tmp  -p 8080:8080 6b2bd203cc3f`来启动，即使用-v参数加了宿主目录与容器目录的映射关系，那么会在/Users/shakeli/idea_workspace/learn-docker/test的目录下，同步容器中/tmp目录中的内容（如果test目录不存在会自动生成）：

![1552494286526.jpg](https://upload-images.jianshu.io/upload_images/3727888-4cf72c37a3bf8111.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 生成正常Web工程的Docker镜像

经过上面的分析，我们可以分析如果只是正常的web工程，我们的Dockerfile最少需要哪些指令。由于springboot集成了Tomcat，而如果不用springboot，那么我们需要加上Tomcat，理论上来说如果从0构建，那么需要：操作系统+JDK/JRE+Tomcat。

由于JDK/JRE需要操作系统，而Tomcat又需要JDK/JRE，那么如果我们以现有的官方镜像为基础，实际上只需要Tomcat的作为基础镜像即可，当然，如果不使用现有的Tomcat镜像，而是以操作系统镜像为基础，加入JDK/JRE，加入Tomcat，会比使用现有Tomcat作为基础镜像少一层。

未完待续……

### 附Dockerfile常用指令

附Dockerfile的常用指令
`FROM <image>:<tag>` 指定基础镜像，必须在Dockerfile第一行
` MAINTAINER <name>`指定维护者信息
` RUN <command>`在镜像中要执行的命令
`WORKDIR`：指定当前工作目录，相当于 cd
`EXPOSE <port> [<port>...]`指定容器要打开的端口
`ENV <key> <value> `或者`ENV <key>=<value> ...`指定一个/多个环境变量，会被后续 RUN 指令使用，并在容器运行时保持
`ADD <src> <dest>`把文件复制到镜像中，src可以是链接
`COPY <src>... <dest>`COPY的<src>只能是本地文件，其他用法与ADD一致
`CMD`或`ENTRYPOINT`容器启动时执行的命令，两者有些许区别，可以组合使用
`ARG <name>[=<default value>]` 设置变量

### IDEADocker插件

未完待续……

### 参考

[Docker —— 从入门到实践](https://yeasy.gitbooks.io/docker_practice/content/)
[Docker命令大全](http://www.runoob.com/docker/docker-run-command.html)
[Dockerfile命令详解（超全版本）](https://www.cnblogs.com/dazhoushuoceshi/p/7066041.html)