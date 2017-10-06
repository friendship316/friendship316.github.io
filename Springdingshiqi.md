---
title: Java定时器，Spring定时器极其部署方式
date: 2017-01-09
---
* 记录几种定时器的实现方式——仅仅在应用层面，简单的实现。本文在完成过程中参考了[详解java定时任务](http://www.cnblogs.com/chenssy/p/3788407.html)、[Spring定时任务的几种实现](http://gong1208.iteye.com/blog/1773177)等文章。

---
###  实现定时器的几种方式
1.  java.util.Timer与java.util.TimerTask。个人总结了一下，这种方式因为有一些缺陷，适合一些简单的定时任务。参考文章[详解java定时任务](http://www.cnblogs.com/chenssy/p/3788407.html)，非常详细。
2. Spring与Quartz的结合，配置比较麻烦，公司的老项目还在用这种方式。
3. Spring3.0以上版本自带的task，配置简单，新项目使用的这种方式。

### 两种部署方式（linux，项目使用spring的情况下）
1. 如果不是web工程，编写一个启动类，用于加载spring的配置文件启动spring容器，并编写启动java程序的脚本（参考[不错的linux下通用的java程序启动脚本](http://www.cnblogs.com/duanxz/p/5581438.html)）。
2. 如果是web工程，将spring加进web.xml的监听器，随着web工程的启动而启动。

###  实例
#### 1\. java.util.Timer与java.util.TimerTask简单实例。
(1) 、创建任务类
```java
  package org.my.timertask;
  import java.text.SimpleDateFormat;
  import java.util.Date;
  import java.util.TimerTask;
  /**
   * 创建任务类，继承TimerTask,重写其run方法，方法体指定要执行的任务。
   * @author fasen
   *
   */
  public class TimerTaskDemo extends TimerTask{
	@Override
	public void run() {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
		System.out.println("这是java.util.TimeTask在执行任务,本次执行时间是："+sdf.format(new Date()));
	}
}
```
(2)、使用调度对象进行调用
```java
  package org.my.timertask;
  import java.util.Timer;
  public class TimerDemo {
	public static void main(String[] args) {
		//创建Timer对象，并调用schedule方法（有四个重载方法）
		Timer t = new Timer();
		//传递参数。第一个参数是TimerTask对象（指定要执行的任务）,第二个参数是延迟时间，第三个参数是时间间隔（毫秒）
		t.schedule(new TimerTaskDemo(),0,5000);
	}
  }
```
运行结果：
![运行结果](http://7xrt1z.com1.z0.glb.clouddn.com/7%60NPRRW5LO$@~YGF@B%5BMP7I.png)
#### 2\. Spring与Quartz的结合使用
 (1)、导入jar包：org.quartz-scheduler，我使用的是2.1.1。我用的Spring的版本是Spring4.0.2，如果使用quartz2.x与Spring3.1以上的版本，配置的bean类必须与本配置一致。根据Spring和quart的版本不同，会有不同的配置。quartz2.x与Spring3.1以上的版本将老版本很多类名加了Factory，比如将CronTriggerBean修改为CronTriggerFactoryBean、将JobDetailBean修改为JobDetailFactoryBean等等，如果用quartz2.x与Spring3.1以上的版本的jar，而使用低版本的配置——CronTriggerBean和JobDetailBean话，会出现类似以下的错误
```java
class org.springframework.scheduling.quartz.JobDetailBean has interface org.quartz.JobDetail as super class
```
这个错误，即代表jar包版本和配置不一致。
```html
<properties>  
      <!-- spring版本号 -->  
      <spring.version>4.0.2.RELEASE</spring.version> 
      <failOnMissingWebXml>false</failOnMissingWebXml>
  </properties> 
<!-- spring核心包 --> 
<dependency>
	    <groupId>org.quartz-scheduler</groupId>
	    <artifactId>quartz</artifactId>
	    <version>2.1.1</version>
	</dependency>
 <!-- spring核心包 -->  
    <dependency>  
        <groupId>org.springframework</groupId>  
        <artifactId>spring-core</artifactId>  
        <version>${spring.version}</version>  
    </dependency> 
```
(2)、编写任务类，Spring与Quartz的结合使用，其任务类有两种方式——任务类继承QuartzJobBean类、任务类不继承特定类。
   >任务类继承QuartzJobBean类，需要重写executeInternal方法：
```java
package org.my.SpringQuartz;
import java.text.SimpleDateFormat;
import java.util.Date;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.scheduling.quartz.QuartzJobBean;
public class SpringQuartzJob extends QuartzJobBean {
	@Override
	protected void executeInternal(JobExecutionContext arg0) throws JobExecutionException {	
		SimpleDateFormat sdf  = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		System.out.println("这是SpringQuartzJob定时任务...任务类继承QuartzJobBean,当前时间："+sdf.format(new Date()));	
	}
}```
> 任务类不继承特定类，POJO，方法名自定义：
```java
package org.my.SpringQuartz;
import java.text.SimpleDateFormat;
import java.util.Date;
public class NotExtendSpringQuartzJob {
	public void dosomething(){
		SimpleDateFormat sdf  = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		System.out.println("这是NotExtendSpringQuartzJob定时任务...任务类不继承特定类，是POJO,现在时间："+sdf.format(new Date()));
	}
}
```
 (3)、spring配置：
```html
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"  
    xmlns:context="http://www.springframework.org/schema/context"  
    xmlns:mvc="http://www.springframework.org/schema/mvc"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans    
                        http://www.springframework.org/schema/beans/spring-beans-4.0.xsd    
                        http://www.springframework.org/schema/context    
                        http://www.springframework.org/schema/context/spring-context-4.0.xsd    
                        http://www.springframework.org/schema/mvc    
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd">  
<!-- 定时任务配置（任务类继承特定的类） 开头 -->
    <!-- 定时任务bean,任务执行类 -->
    <bean name="springQuartzJob" class="org.springframework.scheduling.quartz.JobDetailFactoryBean" >  
		<property name="jobClass" value="org.my.SpringQuartz.SpringQuartzJob" />  
		<!-- 任务类的属性map,可以设置参数，我这里没有属性 -->
		<!-- <property name="jobDataAsMap">  
			<map>  
			<entry key="timeout" value="0" />  
			</map>  
		</property>   -->
	</bean> 
	<!-- 任务执行类对应的触发器：第一种（按照一定的时间间隔触发） -->
	<bean name="springQuartzJobTrigger1" class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
		<property name="jobDetail" ref="springQuartzJob" />  
		<property name="startDelay" value="0" /><!-- 调度工厂实例化后，经过0秒开始执行调度 -->  
		<property name="repeatInterval" value="30000" /><!-- 每30秒调度一次 -->  
	</bean>
	<!-- 任务执行类对应的触发器：第二种（按照指定时间触发，如每天12点） -->
	<bean name="springQuartzJobTrigger2" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
		<property name="jobDetail" ref="springQuartzJob" />  
		<!-- 每天21:37运行一次 -->  
		<property name="cronExpression" value="0 37 21 * * ?" />  
	</bean>
	<!-- 调度工厂 -->
	<bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">  
		<property name="triggers">  
		<list>  
			<ref bean="springQuartzJobTrigger1" />  
			<ref bean="springQuartzJobTrigger2" />
		</list>  
		</property>  
	</bean> 
<!-- 定时任务配置（任务类继承特定的类） 结尾 -->	
<!-- 定时任务配置（任务类不继承特定的类）开头 -->
	<!-- 定时任务bean,任务执行类 -->
	 <bean id="notExtendSpringQuartzJob"  class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">  
		<property name="targetObject">  <!-- 指定任务类 -->
			<bean class="org.my.SpringQuartz.NotExtendSpringQuartzJob" />  
		</property>  
		<property name="targetMethod" value="dosomething" />  <!-- 制定任务类中的方法 -->
		<property name="concurrent" value="false" /><!-- 作业不并发调度 -->  
	</bean> 
	<!-- 任务执行类对应的触发器：第一种（按照一定的时间间隔触发） -->
	<bean name="notExtendSpringQuartzJobTrigger1" class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
		<property name="jobDetail" ref="notExtendSpringQuartzJob" />  
		<property name="startDelay" value="0" /><!-- 调度工厂实例化后，经过0秒开始执行调度 -->  
		<property name="repeatInterval" value="30000" /><!-- 每30秒调度一次 -->  
	</bean>
	<!-- 任务执行类对应的触发器：第二种（按照指定时间触发，如每天12点） -->
	<bean name="notExtendSpringQuartzJobTrigger2" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
		<property name="jobDetail" ref="notExtendSpringQuartzJob" />  
		<!-- 每天21:37运行一次 -->  
		<property name="cronExpression" value="0 37 21 * * ?" />  
	</bean>
	<!-- 调度工厂 -->
	<bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">  
	    <property name="triggers">  
	    <list>  
	   		<ref bean="notExtendSpringQuartzJobTrigger1" />  
			<ref bean="notExtendSpringQuartzJobTrigger2" />  
	    </list>  
	    </property>  
    </bean>
<!-- 定时任务配置（任务类不继承特定的类）结尾 --> 
</beans> 
```
(4)、启动。
```java
package org.my.SpringQuartz;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class StartSpringQuartz {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");	
	}
}
```
eclipse执行结果如下，红框外是根据指定时间间隔触发的——SimpleTriggerFactoryBean，红框内的是设定指定时间触发——CronTriggerFactoryBean，此处配置的是21:37分。![执行结果](http://7xrt1z.com1.z0.glb.clouddn.com/StartSpringQuartz.png)  
#### 3\. Spring3.0以上版本自带的task。
**配置文件方式：**
(1)、编写任务类，POJO。
```java
package org.my.SpringTask;
import java.text.SimpleDateFormat;
import java.util.Date;
import org.springframework.stereotype.Component;
@Component
public class TaskJob {
	public void dosomething(){
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		System.out.println("这是SpringTask定时任务，当前时间："+sdf.format(new Date()));
	}
}
```
(2)、Spring配置（applicationContext-SpringTask.xml），在头部文件中加入了task声明。如果没有声明，会出现类似以下的异常
```java
“org.xml.sax.SAXParseException; lineNumber: 19; columnNumber: 24; cvc-complex-type.2.4.c: 通配符的匹配很全面, 但无法找到元素 'task:scheduled-tasks' 的声明。”```
```html
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"  
    xmlns:context="http://www.springframework.org/schema/context"  
    xmlns:mvc="http://www.springframework.org/schema/mvc"  
    xmlns:task="http://www.springframework.org/schema/task"
    xsi:schemaLocation="http://www.springframework.org/schema/beans    
                        http://www.springframework.org/schema/beans/spring-beans-4.0.xsd    
                        http://www.springframework.org/schema/context    
                        http://www.springframework.org/schema/context/spring-context-4.0.xsd    
                        http://www.springframework.org/schema/mvc    
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
                        http://www.springframework.org/schema/task
                        http://www.springframework.org/schema/task/spring-task-4.0.xsd">  
    <!-- 自动扫描(并启用注解)Spring-task定时方式可以用注解 -->  
    <context:component-scan base-package="org.my.SpringTask" />  

	<!-- Spring task定时配置 -->
	<task:scheduled-tasks>   
		 <!-- 每15秒执行一次，此处可设置为以指定时间间隔或者指定的时间执行，具体由cron表达式决定 -->
	        <task:scheduled ref="taskJob" method="dosomething" cron="0/15 * * * * ?"/>  
	</task:scheduled-tasks> 
</beans> 
```
(3)、启动
```java
package org.my.SpringTask;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class StartSpringTask {

	public static void main(String[] args) {

		ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext-SpringTask.xml");

	}
}
```
运行结果：
![运行结果](http://7xrt1z.com1.z0.glb.clouddn.com/SpringTask.png)
**注解方式：**
(1)、任务类：
```java
package org.my.SpringTaskComment;

  import java.text.SimpleDateFormat;
  import java.util.Date;

  import org.springframework.scheduling.annotation.Scheduled;
  import org.springframework.stereotype.Component;
  @Component
  public class TaskJobComment {
	@Scheduled(cron="0/15 * * * * ?")//每15秒执行一次
	public void dosomething(){
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		System.out.println("这是SpringTask定时任务注解版，当前时  间："+sdf.format(new Date()));
	}
}
```
(2)、Spring开启注解扫描
```html
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"  
    xmlns:context="http://www.springframework.org/schema/context"  
    xmlns:mvc="http://www.springframework.org/schema/mvc"  
    xmlns:task="http://www.springframework.org/schema/task"
    xsi:schemaLocation="http://www.springframework.org/schema/beans    
                        http://www.springframework.org/schema/beans/spring-beans-4.0.xsd    
                        http://www.springframework.org/schema/context    
                        http://www.springframework.org/schema/context/spring-context-4.0.xsd    
                        http://www.springframework.org/schema/mvc    
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
                        http://www.springframework.org/schema/task
                        http://www.springframework.org/schema/task/spring-task-4.0.xsd">  
    <!-- 自动扫描(并启用注解)-->  
    <context:component-scan base-package="org.my" />  
  <!-- 开启这个扫描才能使用定时注解，这个配置项有一些非必需参数（属性），比如executor、scheduler、mode等等--> 
     <task:annotation-driven/>
</beans> 
```  
(3)、启动
```java
package org.my.SpringTaskComment;
 import org.springframework.context.ApplicationContext;
 import org.springframework.context.support.ClassPathXmlApplicationContext;
 public class StartSpringTaskComment {
	public static void main(String[] args) {		
		ApplicationContext ctx = new  ClassPathXmlApplicationContext("applicationContext-SpringTask.xml");
	}
}
``` 
执行结果：
![执行结果](http://7xrt1z.com1.z0.glb.clouddn.com/SpringTaskComment.png)

至此，三种定时方式简单实现了一遍，更多的配置参数，以及效率、性能在此未作深入研究。