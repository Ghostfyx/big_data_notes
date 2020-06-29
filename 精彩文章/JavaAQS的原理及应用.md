# AQS的原理及应用

文章摘自美团技术沙龙——[从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)

## 前言

Java中的大部分同步类(Lock、Semaphore、ReentrantLock等)都是基于AbstractQueuedSynchronizer(简称为AQS)实现的。AQS是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。本文会从应用层逐渐深入到原理层，并通过ReentrantLock的基本特性和ReentrantLock与AQS的关联，来深入解读AQS相关**独占锁**的知识点，同时采取问答的模式来帮助大家理解AQS。由于篇幅原因，本篇文章主要阐述AQS中独占锁的逻辑和Sync Queue，不讲述包含共享锁和Condition Queue的部分(本篇文章核心为AQS原理剖析，只是简单介绍了ReentrantLock，感兴趣同学可以阅读一下ReentrantLock的源码)。

![](img/Java_AQS.jpg)

## 1. ReentrantLock

### 1.1 ReentrantLock特性概览

ReentrantLock意思为可重入锁，指的是**一个线程能够对一个临界资源重复加锁**。为了帮助大家更好地理解ReentrantLock的特性，我们先将ReentrantLock跟常用的Synchronized进行比较，其特性如下（蓝色部分为本篇文章主要剖析的点）：

<img src="img/ReentranctLock_Synchronized.jpg" style="zoom:67%;" />

下面通过伪代码，进行更加直观的比较：

```java
// **************************Synchronized的使用方式**************************
// 1.用于代码块
synchronized (this) {}
// 2.用于对象
synchronized (object) {}
// 3.用于方法
public synchronized void test () {}
// 4.可重入
for (int i = 0; i < 100; i++) {
	synchronized (this) {}
}
// **************************ReentrantLock的使用方式**************************
public void test () throw Exception {
	// 1.初始化选择公平锁、非公平锁
	ReentrantLock lock = new ReentrantLock(true);
	// 2.可用于代码块
	lock.lock();
	try {
		try {
			// 3.支持多种加锁方式，比较灵活; 具有可重入特性
			if(lock.tryLock(100, TimeUnit.MILLISECONDS)){ }
		} finally {
			// 4.手动释放锁
			lock.unlock()
		}
	} finally {
		lock.unlock();
	}
}
```

### 1.2 ReentrantLock与AQS的关联

ReentrantLock支持公平锁和非公平锁，，并且ReentrantLock的底层就是由AQS来实现的。那么ReentrantLock是如何通过公平锁和非公平锁与AQS关联起来呢？ 我们着重从这两者的加锁过程来理解一下它们与AQS之间的关系(加锁过程中与AQS的关联比较明显，解锁流程后续会介绍)。

**非公平锁**源码中的加锁流程如下

```java
// java.util.concurrent.locks.ReentrantLock#NonfairSync

// 非公平锁
static final class NonfairSync extends Sync {
	...
	final void lock() {
		if (compareAndSetState(0, 1))
			setExclusiveOwnerThread(Thread.currentThread());
		else
			acquire(1);
		}
  ...
}
```

这块代码的含义为：

- 若通过CAS设置变量State(同步状态)成功，也就是获取锁成功，则将当前线程设置为独占线程。
- 若通过CAS设置变量State(同步状态)失败，也就是获取锁失败，则进入Acquire方法进行后续处理。

第一步很好理解，但第二步获取锁失败后，后续的处理策略是怎么样的呢？这块可能会有以下思考：

**问题1：**某个线程获取锁失败的后续流程是什么呢？有以下两种可能：

- 将当前线程获锁结果设置为失败，获取锁流程结束。这种设计会极大降低系统的并发度(我的理解是：这样会导致频繁的线程切换，因为业务处理流程中获取锁失败一般意味着请求失败)，并不满足我们实际的需求。所以就需要下面这种流程，也就是AQS框架的处理流程
- 存在某种排队等候机制，线程继续等待，仍然保留获取锁的可能，获取锁流程仍在继续

对于问题1的第二种情况，既然说到了排队等候机制，那么就会引出以下几个问题：

1. 排队等候机制就一定会有某种队列形成，这样的队列是什么数据结构呢？
2. 处于排队等候机制中的线程，什么时候可以有机会获取锁呢？
3. 如果处于排队等候机制中的线程一直无法获取锁，还是需要一直等待吗，还是有别的策略来解决这一问题？



