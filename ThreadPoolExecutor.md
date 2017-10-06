---
title: Java线程池带图源码解析
date: 2017-09-25
tags: 
    - 线程池
    - Java
---
线程池作为Java中一个重要的知识点，看了很多文章，在此以Java自带的线程池为例，记录分析一下。本文参考了[Java并发编程：线程池的使用](http://www.cnblogs.com/dolphin0520/p/3932921.html)、[Java线程池---addWorker方法解析](http://www.jianshu.com/p/3527c91f542d)、[线程池](http://blog.csdn.net/wojiaolinaaa/article/details/51345789)、[ThreadPoolExecutor中策略的选择与工作队列的选择（java线程池）](http://blog.csdn.net/longeremmy/article/details/8231184)和[ThreadPoolExecutor线程池解析与BlockingQueue的三种实现](http://blog.csdn.net/a837199685/article/details/50619311)。本文基于JDK1.8实现用例和源码分析，并主要着重流程。

### 一、使用线程池
要知道一个东西的原理，首先要知道如何使用它。所以先上一个使用线程池的示例。
#### 1、任务类
要使用Java自带的线程池，首先需要一个任务类，这个任务类需要实现Runnable接口，并重写run方法（需要多线程执行的任务逻辑）。
```java
package org.my.threadPoolDemo;
/**
 * 任务类，实现Runnable接口 重写run方法
 */
public class MyTask implements Runnable{
	
	private int taskNum;
	
	public MyTask(int taskNum) {
		super();
		this.taskNum = taskNum;
	}

	@Override
	public void run() {
		System.out.println("正在执行task"+taskNum);
		try {
			Thread.currentThread().sleep(4000);//sleep 4秒模拟执行代码过程
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("task"+taskNum+"执行完毕");
	}
}

```
#### 2、创建线程池并执行多个任务
有了任务类，接下来创建线程池，并执行多个任务。我们使用ThreadPoolExecutor来创建线程池。
```java
package org.my.threadPoolDemo;

import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPoolUseDemo {

	public static void main(String[] args) {
		ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS, new LinkedBlockingDeque<Runnable>());
		for (int i = 0; i < 15; i++) {
			MyTask myTask = new MyTask(i);
			executor.execute(myTask);
			System.out.println("线程池中线程数量："+executor.getPoolSize()+",线程池中等待执行的任务数量："+executor.getQueue().size()+",已执行完的任务数量："+executor.getCompletedTaskCount());
		}
		executor.shutdown();
	}

}
```
逻辑很简单，创建了一个线程池管理器，然后使用它执行了15个任务。这里需要解释一下构造ThreadPoolExecutor的时候传入的参数的意义：
>* 5（corePoolSize）是指核心池大小。即创建的线程数量。如果线程池中线程数等于这个数量，那么下一个来的任务就会被放在任务队列中（稍后详细图解）。
>* 10（maximumPoolSize）是指线程池能创建的最大线程数量。如果上一步的任务队列已满，那么线程池将继续创建线程，直到线程数=10，下一个来的任务会被拒绝执行。
>* 200（keepAliveTime）是指表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0。
>* TimeUnit.MILLISECONDS是keepAliveTime的时间单位。
>```text
>TimeUnit.DAYS;               //天
>TimeUnit.HOURS;             //小时
>TimeUnit.MINUTES;           //分钟
>TimeUnit.SECONDS;           //秒
>TimeUnit.MILLISECONDS;      //毫秒
>TimeUnit.MICROSECONDS;      //微妙
>TimeUnit.NANOSECONDS;       //纳秒
>```
>* ArrayBlockingQueue是传入的阻塞队列，用来存放任务的队列workQueue。即上文中提到的任务队列。除了ArrayBlockingQueue，还有LinkedBlockingQueue、SynchronousQueue可以选择。

ThreadPoolExecutor拥有四个构造器，除了上方传入的参数以外，还有另外的构造器可以传入：

>* threadFactory：线程工厂，主要用来创建线程
>* handler：表示拒绝执行任务时的策略，有四种取值：
>```text
>ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
>ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
>ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
>ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务 
>```

执行上面的程序，结果如下：
```text
正在执行task0
线程池中线程数量：1,线程池中等待执行的任务数量：0,已执行完的任务数量：0
线程池中线程数量：2,线程池中等待执行的任务数量：0,已执行完的任务数量：0
正在执行task1
线程池中线程数量：3,线程池中等待执行的任务数量：0,已执行完的任务数量：0
正在执行task2
线程池中线程数量：4,线程池中等待执行的任务数量：0,已执行完的任务数量：0
正在执行task3
线程池中线程数量：5,线程池中等待执行的任务数量：0,已执行完的任务数量：0
正在执行task4
线程池中线程数量：5,线程池中等待执行的任务数量：1,已执行完的任务数量：0
线程池中线程数量：5,线程池中等待执行的任务数量：2,已执行完的任务数量：0
线程池中线程数量：5,线程池中等待执行的任务数量：3,已执行完的任务数量：0
线程池中线程数量：5,线程池中等待执行的任务数量：4,已执行完的任务数量：0
线程池中线程数量：5,线程池中等待执行的任务数量：5,已执行完的任务数量：0
线程池中线程数量：6,线程池中等待执行的任务数量：5,已执行完的任务数量：0
正在执行task10
线程池中线程数量：7,线程池中等待执行的任务数量：5,已执行完的任务数量：0
正在执行task11
线程池中线程数量：8,线程池中等待执行的任务数量：5,已执行完的任务数量：0
正在执行task12
正在执行task13
线程池中线程数量：9,线程池中等待执行的任务数量：5,已执行完的任务数量：0
线程池中线程数量：10,线程池中等待执行的任务数量：5,已执行完的任务数量：0
正在执行task14
task1执行完毕
task10执行完毕
正在执行task5
task11执行完毕
task13执行完毕
task4执行完毕
task3执行完毕
task2执行完毕
task0执行完毕
正在执行task9
正在执行task8
正在执行task7
task14执行完毕
task12执行完毕
正在执行task6
task5执行完毕
task9执行完毕
task7执行完毕
task8执行完毕
task6执行完毕
```
可以看到，由于corePoolSize是5，所以当任务数量大于5的时候，接下来来的任务放入了任务队列等待，但是由于任务队列最大容量是5，maximumPoolSize=10，所以在任务队列满了之后，线程池管理器又继续创建了5个线程，最终线程池中线程数量达到了10。这时候15个任务能够由这些线程处理完，如果再增加任务，比如将for循环次数增加到20，就出现了java.util.concurrent.RejectedExecutionException。
### 二、原理分析
从上面使用线程池的例子来看，最主要就是两步，构造ThreadPoolExecutor对象，然后每来一个任务，就调用ThreadPoolExecutor对象的execute方法。
#### 1、ThreadPoolExecutor结构
ThreadPoolExecutor的主要结构及继承关系如下图所示：

![ThreadPoolExecutor结构及继承关系](http://owrjz8s8v.bkt.clouddn.com/ThreadPoolExecutor.png)

主要成员变量：任务队列——存放那些暂时无法执行的任务；工作线程池——存放当前启用的所有线程；线程工厂——创建线程；还有一些用来调度线程与任务并保证线程安全的成员。

了解了ThreadPoolExecutor的主要结构，再简单梳理一下“一个传入线程池的任务能够被最终正常执行需要经过的主要流程”，方法名称前面没有“XXX.”这种标注的都是ThreadPoolExecutor的方法：

![线程池主要流程](http://owrjz8s8v.bkt.clouddn.com/ThreadPoolliucheng.jpg)

#### 2、ThreadPoolExecutor构造器及重要常量
简单了解下构造器，ThreadPoolExecutor的四个构造器的源码如下：
```java
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,Executors.defaultThreadFactory(), defaultHandler);
    }
    
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,threadFactory, defaultHandler);
    }
    
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,Executors.defaultThreadFactory(), handler);
    }
    
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||maximumPoolSize <= 0 ||maximumPoolSize < corePoolSize ||keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?null :AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
从源码中可以看出，这四个构造器都是调用最后一个构造器，只是根据开发者传入的参数的不同而填充一些默认的参数。比如如果开发者没有传入线程工厂threadFactory参数，那么构造器就使用默认的Executors.defaultThreadFactor。

在这里还要理解ThreadPoolExecutor的几个常量的含义和几个简单方法：
```java
//Integer.SIZE是一个静态常量，值为32，也就是说COUNT_BITS是29
private static final int COUNT_BITS = Integer.SIZE - 3;
//CAPACITY是最大容量536870911，因为1左移29位之后-1，导致最高三位为0，而其余29位全部为1
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
//ctl用于表示线程池状态和有效线程数量，最高三位表示线程池的状态，其余低位表示有效线程数量
//初始化之后ctl等于RUNNING的值，即默认状态是运行状态，线程数量为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//1110 0000 0000 0000 0000 0000 0000 0000最高三位为111
private static final int RUNNING    = -1 << COUNT_BITS;
//最高三位为000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//0010 0000 0000 0000 0000 0000 0000 0000最高三位为001
private static final int STOP       =  1 << COUNT_BITS;
//0100 0000 0000 0000 0000 0000 0000 0000最高三位为010
private static final int TIDYING    =  2 << COUNT_BITS;
//0110 0000 0000 0000 0000 0000 0000 0000最高三位为011
private static final int TERMINATED =  3 << COUNT_BITS;
/**
* 获取运行状态，入参为ctl。因为CAPACITY是最高三位为0，其余低位为1
* 所以当取反的时候，就只有最高三位为1，再经过与运算，只会取到ctl的最高三位
* 而这最高三位如上文所述，表示线程池的状态
*/
private static int runStateOf(int c)     { return c & ~CAPACITY; }
/**
* 获取工作线程数量，入参为ctl。因为CAPACITY是最高三位为0，其余低位为1
* 所以当进行与运算的时候，只会取到低29位，这29位正好表示有效线程数量
*/
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
//任务队列，用于存放待执行任务的阻塞队列
private final BlockingQueue<Runnable> workQueue;
/** 判断线程池是否是运行状态，传入的参数是ctl的值
* 只有RUNNING的符号位是1，也就是只有RUNNING为负数
* 所以如果目前的ctl值<0，就是RUNNING状态
*/
private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
//从任务队列中移除任务
public boolean remove(Runnable task) {
    boolean removed = workQueue.remove(task);
    tryTerminate(); // In case SHUTDOWN and now empty
    return removed;
}
```
#### 3、getTask()源代码
通过前面的流程图，我们知道getTask()是由runWorker方法调用的，目的是取出一个任务。
```java
private Runnable getTask() {
        //记录循环体中上个循环在从阻塞队列中取任务时是否超时
        boolean timedOut = false; 
        //无条件循环，主要是在线程池运行正常情况下
        //通过循环体内部的阻塞队列的阻塞时间，来控制当前线程的超时时间
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);//获取线程池状态

            /*先获取线程池的状态，如果状态大于等于STOP，也就是STOP、TIDYING、TERMINATED之一
             *这时候不管队列中有没有任务，都不用去执行了；
             *如果线程池的状态为SHUTDOWN且队列中没有任务了，也不用继续执行了
             *将工作线程数量-1，并且返回null
             **/
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            //获取工作线程数量
            int wc = workerCountOf(c);

            //是否启用超时参数keepAliveTime
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            //如果这个条件成立，如果工作线程数量-1成功，返回null，否则跳出循环
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
            //如果需要采用阻塞形式获取，那么就poll设定阻塞时间，否则take无限期等待。
            //这里通过阻塞时间，间接控制了超时的值，如果取值超时，意味着这个线程在超时时间内处于空闲状态
            //那么下一个循环，将会return null并且线程数量-1
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
#### 4、runWorker(Worker w)源代码
通过上面流程图，可以看出runWorker(Worker w)实际上已经是在线程启动之后执行任务了，所以其主要逻辑就是获取任务，然后执行任务的run方法。
```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;//获取传入的Worker对象中的任务
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
        		/* 
        		 * 如果传入的任务为null，就从任务队列中获取任务，只要这两者有一个不为null，就进入循环体
        		 */
            while (task != null || (task = getTask()) != null) {
                w.lock();
                //如果线程池状态已被标为停止，那么则不允许该线程继续执行任务！或者该线程已是中断状态,
                //也不允许执行任务,还需要中断该线程!
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);//为子类预留的空方法（钩子）
                    Throwable thrown = null;
                    try {
                        task.run();//终于真正执行传入的任务了
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);//为子类预留的空方法（钩子）
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
                /*
                 * 从这里可以看出，当一个任务执行完毕之后，循环并没有退出，此时会再次执行条件判断
	             * 也就是说如果执行完第一个任务之后，任务队列中还有任务，那么将会继续在这个线程执行。
	             * 这里也是很巧妙的地方，不需要额外开一个控制线程来看那些线程处于空闲状态，然后给他分配任务。
	             * 直接自己去任务队列拿任务
                 */
            }
            completedAbruptly = false;
        } finally {
        		/*
        		 * 执行到这里，说明getTask返回了null，要么是超时（任务队列没有任务了），要么是线程池状态有问题了
        		 * 当前线程将被回收了
        		 */
            processWorkerExit(w, completedAbruptly);
        }
    }
```
#### 5、addWorker(Runnable firstTask, boolean core)源代码
runWorker是从Worker对象中获取第一个任务，然后从任务队列中一直获取任务执行。流程图中已经说过，这个Worker对象是在addWorker方法中创建的，所以新线程创建、启动的源头是在addWorker方法中。而addWorker是被execute所调用，execute根据addWorker的返回值，进行条件判断。
```java
private boolean addWorker(Runnable firstTask, boolean core) {
		//上来就是retry，后面continue retry;语句执行之后都会从这里重新开始
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);//获取线程池运行状态

            /*
             * 获取当前线程池的状态，如果是STOP，TIDYING,TERMINATED状态的话，则会返回false
             * 如果现在状态是SHUTDOWN，但是firstTask不为空或者workQueue为空的话，那么直接返回false
             */
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);//获取工作线程数量
                /*
                 * addWorker传入的第二个Boolean参数用来判别当前线程数量是否大于核心池数量
                 * true，代表当前线程数量小于核心池数量
                 */
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //增加工作线程数量
                if (compareAndIncrementWorkerCount(c))
                		//如果成功，跳出retry
                    break retry;
                c = ctl.get();  // Re-read ctl
                //判断线程池状态，改变了就重试一次
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
        		/*
        		 * 前面顺利增加了工作线程数，那么这里就真正创建Worker
        		 * 上面流程图中说过，创建Worker时会创建新的线程.如果这里创建失败
        		 * finally中会将工作线程数-1
        		 */
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);//将Worker放入工作线程池
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();//启动线程
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
#### 6、execute方法
execute方法其实最主要是根据线程池的策略来传递不同的参数给addWorker方法。
当调用 execute() 方法添加一个任务时，线程池会做如下判断：
      
  a. 如果正在运行的线程数量小于 corePoolSize，那么马上创建线程运行这个任务；
 b. 如果正在运行的线程数量大于或等于 corePoolSize，那么将这个任务放入队列。
c. 如果这时候队列满了，而且正在运行的线程数量小于 maximumPoolSize，那么还是要创建线程运行这个任务；
d. 如果队列满了，而且正在运行的线程数量大于或等于 maximumPoolSize，那么线程池会抛出异常，告诉调用者“我不能再接受任务了”。

方法execute主要是在控制上面四条策略的实现。
```java
 public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        //获取到成员变量ctl的值（线程池状态）
        int c = ctl.get();
        //如果工作线程的数量<核心池的大小
        if (workerCountOf(c) < corePoolSize) {
            //调用addWorker（这里传入的true代表工作线程数量<核心池大小）
            //如果成功，方法结束。
            if (addWorker(command, true))
                return;
            //否则，再重新获取一次ctl的值
            //个人理解是防止前面这段代码执行的时候有其他线程改变了ctl的值。
            c = ctl.get();
        }
        //如果工作线程数量>=核心池的大小或者上一步调用addWorker返回false，继续走到下面
        //如果线程池处于运行状态，并且成功将当前任务放入任务队列
        if (isRunning(c) && workQueue.offer(command)) {
            //为了线程安全，重新获取ctl的值
            int recheck = ctl.get();
            //如果线程池不处于运行状态并且任务从任务队列移除成功
            if (! isRunning(recheck) && remove(command))
                //调用reject拒绝执行，根据handler的实现类抛出异常或者其他操作
                reject(command);
            //否则，如果工作线程数量==0，调用addWorker并传入null和false
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //执行到这里代表当前线程已超越了核心线程且任务提交到任务队列失败。（可以注意这里的addWorker是false）
        //那么这里再次调用addWroker创建新线程（这时创建的线程是maximumPoolSize）。
        //如果还是提交任务失败则调用reject处理失败任务
        else if (!addWorker(command, false))
            reject(command);
    }
```
### 三、线程池相关影响因素
#### 1、阻塞队列的选择
本段引用至[ThreadPoolExecutor中策略的选择与工作队列的选择（java线程池）](http://blog.csdn.net/longeremmy/article/details/8231184)

**直接提交：**
工作队列的默认选项是SynchronousQueue，它将任务直接提交给线程而不存储它们。在此，如果不存在可用于立即运行任务的线程，则试图把任务加入队列将失败，因此会构造一个新的线程。此策略可以避免在处理可能具有内部依赖性的请求集时出现锁。直接提交通常要求无界maximumPoolSize以避免拒绝新提交的任务。当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。

**无界队列：**
使用无界队列（例如，不具有预定义容量的LinkedBlockingQueue将导致在所有 corePoolSize 线程都忙时新任务在队列中等待。这样，创建的线程就不会超过corePoolSize。（因此，maximumPoolSize的值也就无效了。）当每个任务完全独立于其他任务，即任务执行互不影响时，适合于使用无界队列；例如，在 Web 页服务器中。这种排队可用于处理瞬态突发请求，当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。

**有界队列：**
当使用有限的maximumPoolSize时，有界队列（如 ArrayBlockingQueue）有助于防止资源耗尽，但是可能较难调整和控制。队列大小和最大池大小可能需要相互折衷：使用大型队列和小型池可以最大限度地降低 CPU 使用率、操作系统资源和上下文切换开销，但是可能导致人工降低吞吐量。如果任务频繁阻塞（例如，如果它们是 I/O 边界），则系统可能为超过您许可的更多线程安排时间。使用小型队列通常要求较大的池大小，CPU 使用率较高，但是可能遇到不可接受的调度开销，这样也会降低吞吐量。

#### 2、常用的线程池
在使用ThreadPoolExecutor创建线程池的时候，根据不同参数，可以使用不同构造器构造不同特性的线程池。而实际情况中，我们一般使用Executors类提供的方法来创建线程池。Executors最终也是调用ThreadPoolExecutor的构造器，但是已经配置好了相关参数。

1）固定大小的线程池：Executors.newFixedThreadPool
>corePoolSize和maximumPoolSize相同，超时时间为0，队列用的LinkedBlockingQueue无界的FIFO队列，这表示这个线程池始终只有corePoolSize个线程在运行。根据前面分析的，如果执行完任务，这个线程会继续从任务队列取任务执行，当没有任务的时候，线程会立即关闭。

2）单任务线程池：Executors.newSingleThreadExecutor
>池中只有一个线程工作，阻塞队列无界，它能保证按照任务提交的顺序来执行任务。

3）可变尺寸的线程池：Executors.newCachedThreadPool
>SynchronousQueue队列，一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作。所以，当我们提交第一个任务的时候，是加入不了队列的，这就满足了，一个线程池条件“当无法加入队列的时候，且任务没有达到maximumPoolSize时，我们将新开启一个线程任务”。所以我们的maximumPoolSize是ctl低29位的最大值。超时时间是60s，当一个线程没有任务执行会暂时保存60s超时时间，如果没有的新的任务的话，会从线程池中remove掉。

4）scheduled线程池，Executors.newScheduledThreadPool
>创建一个大小无限的线程池。此线程池支持定时以及周期性执行任务的需求

终于差不多可以画上句号了，感觉有很多东西还是没有细说，尤其是其中大量的重入锁。有机会再补上。综合了很多网上资源，也自己画了几张图，个人认为相对于很多介绍线程池的博客来说，更容易理解吧。如有错误，欢迎批评指正。
