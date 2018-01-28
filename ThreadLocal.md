---
title: 图说ThreadLocal
date: 2018-01-28 22:23:00
---
对ThreadLocal的实现一直比较模棱两可，于是简单总结一下。若有错误，欢迎指出，共同进步~。

众所周知，ThreadLocal的设计原理是给每个线程保存一份独立的变量的副本。这样就可以避免因为多个线程对同一份资源（变量）的操作而产生一些并发问题。相比于同步锁来说，是空间换时间的做法。在许多框架中，应用非常广泛。比如Spring的Singleton作用域模式，以及Struts2的ActionContext等等。

###简单使用样例
要理解一样东西，首先要学会使用，还是老的思路，先写个基本的使用的栗子。
这个栗子中，ThreadLocalDemo 拥有一个ThreadLocal类型的静态变量local，然后在main方法中启动了两个线程，这两个线程分别对local中的数据（一个Integer类型的变量）进行初始化，然后进行加的操作，在操作的过程中使用了Thread.sleep(100L)增大两个线程交叉执行的几率。
```Java
public class ThreadLocalDemo {
	private static ThreadLocal<Integer> local = new ThreadLocal<Integer>();

	public static void main(String[] args) {
		Runnable r1 = new Runnable() {
			@Override
			public void run() {
				local.set(0);
				Integer n = local.get();
				for (int i = 0; i < 10; i++) {
					try {
						Thread.sleep(100L);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("线程r1:" + n++);
					local.set(n);
				}
			}
		};
		Runnable r2 = new Runnable() {
			@Override
			public void run() {
				local.set(100);
				Integer n = local.get();
				for (int i = 0; i < 10; i++) {
					try {
						Thread.sleep(100L);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("线程r2:" + n++);
					local.set(n);
				}
			}
		};
		Thread t1 = new Thread(r1);
		Thread t2 = new Thread(r2);
		t1.start();
		t2.start();
	}

}
```
运行结果：
```text
线程r1:0
线程r2:100
线程r1:1
线程r2:101
线程r1:2
线程r2:102
线程r1:3
线程r2:103
线程r1:4
线程r2:104
线程r1:5
线程r2:105
线程r1:6
线程r2:106
线程r1:7
线程r1:8
线程r2:107
线程r1:9
线程r2:108
线程r2:109
```
从结果看到，两个线程的操作互不干扰，最后的输出结果与各自线程单独执行的结果一致。这就是因为两个线程各自保存了这个变量的副本。此处如果在main方法中设置初始值（local.set(0)）而不在各自的线程里设置，那么在r1和r2中取值时会报空指针异常，进一步说明了这个问题：ThreadLocal中的变量是存在当前线程中的。
本例中是将一个Integer存入到ThreadLocal中，如果我们将这个Integer换成一个map，那么我们就可以在一个ThreadLocal存放很多我们自己定义的数据了。

###原理解析
####宏观关系分析
先上个图就一目了然了，根据这个图做一个简单的分析。

![threadLocal结构关系](http://upload-images.jianshu.io/upload_images/3727888-cb55f4e396dedc0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图中左边是ThreadLocal类，右边就是Thread类，也就是线程类。从宏观上来说：
ThreadLocal中拥有一个内部类ThreadLocalMap、一个set方法、一个get方法。
ThreadLocalMap是中是一个Entry结构（键值对，可以类比HashMap），这个Entry的key是一个ThreadLocal对象。
Thread类中声明了一个ThreadLocalMap类型的变量。
ThreadLocal中的set方法、get方法实际上是在操作Thread类中的threadLocals。
####源码分析
#####ThreadLocal的set、get方法
set方法:
```Java
public void set(T value) {
    //获取当前线程的ThreadLocalMap，如果不为空
    //以当前ThreadLocal对象为key，将当前value存进去
    //如果为空，就创建一个新的map
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
//ThreadLocalMap的set方法
private void set(ThreadLocal<?> key, Object value) {
    //创建Entry数组，计算hashcode值
    //如果hashcode对应的位置已经有Entry值，那么查看下一个位置，直到找到空的位置的下标
    //这里有一个细节，如果找到一个位置有Entry值，但是其Entry的key为null（说明其ThreadLocal对象已经被回收，后面会讲到出现这种状况的原因）
    //这个时候会调用replaceStaleEntry进行一个替换的操作，将当前key/value存入进去
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
get方法：
```Java
public T get() {
    //获取当前线程的ThreadLocalMap
    //如果不为空，以当前ThreadLocal对象为key，获取value值
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //如果map为null，设置初始值
    return setInitialValue();
}
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
#####ThreadLocalMap
在set方法的时候，有一个检查ThreadLocalMap中键值对的key为null，但是本身Entry对象不为null的细节，为了明白这个细节，最后我们看ThreadLocalMap的部分源码：
```java
static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        ……
}
```
这里ThreadLocalMap中的Entry继承了WeakReference，WeakReference是用来做什么的呢？
下面这段话摘自《深入理解Java虚拟机》：
>无论是通过引用计数算法判断对象的引用数量， 还是通过可达性分析算法判断对象的引用链是否可达， 判定对象是否存活都与“引用”有关。 在JDK 1.2以前， Java中的引用的定义很传统： 如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址， 就称这块内存代表着一个引用。 这种定义很纯粹， 但是太过狭隘， 一个对象在这种定义下只有被引用或者没有被引用两种状态， 对于如何描述一些“食之无味， 弃之可惜”的对象就显得无能为力。 我们希望能描述这样一类对象： 当内存空间还足够时， 则能保留在内存之中； 如果内存空间在进行垃圾收集后还是非常紧张， 则可以抛弃这些对象。 很多系统的缓存功能都符合这样的应用场景。在JDK 1.2之后， Java对引用的概念进行了扩充， 将引用分为强引用（ StrongReference） 、 软引用（ Soft Reference） 、 弱引用（ Weak Reference） 、 虚引用（ PhantomReference） 4种， 这4种引用强度依次逐渐减弱。强引用就是指在程序代码之中普遍存在的， 类似“Object obj=new Object（ ） ”这类的引用， 只要强引用还存在， 垃圾收集器永远不会回收掉被引用的对象。软引用是用来描述一些还有用但并非必需的对象。 对于软引用关联着的对象， 在系统将要发生内存溢出异常之前， 将会把这些对象列进回收范围之中进行第二次回收。 如果这次回收还没有足够的内存， 才会抛出内存溢出异常。 在JDK 1.2之后， 提供了SoftReference类来实现软引用。弱引用也是用来描述非必需对象的， 但是它的强度比软引用更弱一些， 被弱引用关联的对象只能生存到下一次垃圾收集发生之前。 当垃圾收集器工作时， 无论当前内存是否足够，都会回收掉只被弱引用关联的对象。 在JDK 1.2之后， 提供了WeakReference类来实现弱引用。虚引用也称为幽灵引用或者幻影引用， 它是最弱的一种引用关系。 一个对象是否有虚引用的存在， 完全不会对其生存时间构成影响， 也无法通过虚引用来取得一个对象实例。 为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。 在JDK 1.2之后， 提供了PhantomReference类来实现虚引用。

通过这段话我们知道，WeakReference是用来将这个类声明为弱引用的，一个类如果声明为弱引用类型，那么这个类的实例会在下一次垃圾回收的时候被回收。那么这里的Entry为什么要特殊设置便于垃圾回收呢？
下图是本文开头的示例中ThreadLocal对象被引用的关系图：
![ThreadLocal引用关系](http://upload-images.jianshu.io/upload_images/3727888-927c8c7bb5f23ea5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
以本文开头的用例来说，ThreadLocalDemo中有一个ThreadLocal对象的引用local，而在线程t1、t2中的ThreadLocalMap中的key也指向了这个ThreadLocal对象，如果我在main线程中不需要这个ThreadLocal对象了，我人为将静态变量local指向为null，那么这个ThreadLocal对象应该被回收，因为强引用已经不在了。但是此时线程t1、t2中的ThreadLocalMap中的key还引用了这个ThreadLocal对象，如果这个引用是强引用，那么很显然ThreadLocal对象将永远不会被回收，直到t1、t2线程终止，如果t1、t2是线程池中的线程，长期不会被终止，那么这个ThreadLocal对象也就无法被回收，产生内存泄漏。而弱引用就很好的解决了这个问题，因为这个ThreadLocal对象的强引用一旦消失，只剩下几个弱引用，那么下次垃圾回收的时候，这个ThreadLocal对象还是会被回收。**所以就会出现上文set的时候，有的位置有Entry对象，但是却没有key的情况。**
参考：[十分钟理解Java中的弱引用](http://www.importnew.com/21206.html)、[ThreadLocal可能引起的内存泄露](http://www.cnblogs.com/onlywujun/p/3524675.html)


