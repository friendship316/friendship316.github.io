---
title: wait与notify的使用
date: 2016-12-27
---
简单记录一个Demo（两个线程轮流打印数字，一个打印奇数，一个打印偶数，每打印10个就交换）
```java
package org.my.threadPoolDemo;

import java.util.concurrent.CountDownLatch;

public class WaitNotify {

	private static final Object syc = new Object();
	
	public static CountDownLatch lat = new CountDownLatch(2);
	
	public static void main(String[] args) {
		
		Runnable R1 = new Runnable() {
			@Override
			public void run() {
					int count = 0;
					for(int i = 0;i<100;i++) {
						if(i%2 == 0) {
							System.out.println("偶数："+i);
							count++;
						}
						if(count == 10) {
							try {
								synchronized (syc) {
									syc.notify();
									syc.wait();
								}
								count = 0;
							} catch (InterruptedException e) {
								e.printStackTrace();
							}
						}
					}
					System.out.println("偶数结束");
					synchronized (syc) {
						syc.notify();
					}
					lat.countDown();
				
			}
		};
		
		Runnable R2 = new Runnable() {
			@Override
			public void run() {
					int count = 0;
					for(int i=0;i<100;i++) {
						if(i%2 == 1) {
							System.out.println("奇数："+i);
							count++;
						}
						if(count == 10) {
							try {
								synchronized (syc) {
									syc.notify();
									syc.wait();
								}
								count = 0;
							} catch (InterruptedException e) {
								e.printStackTrace();
							}
						}
					}
					System.out.println("奇数结束");
					synchronized(syc) {
						syc.notify();
					}
					lat.countDown();
			}
		};

		Thread t1 = new Thread(R1);
		Thread t2 = new Thread(R2);
		
		t1.start();
		t2.start();
		
		try {
			lat.await();
			System.out.println("两个线程全部结束");
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

}

```