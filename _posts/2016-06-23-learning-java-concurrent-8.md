---
date: 2016-06-23 22:16:12+08:00
layout: post
title: 一起学java并发编程（八）：JMM
thread: 10
categories: java
tags: java并发编程
---
### 存储模型
Java语言规范规定了JVM要维护内部线程类似顺序化语意：只要程序的最终结果等同于它在严格的顺序化环境中执行的结果，那么上述所有的行为都是允许的。重新排序后的指令使得程序在计算性能上得到了很大的提升。JMM规定了JVM的一种最小保证：什么时候写入一个变量会对其他线程可见。这样的设计，可以在对可预言性的需要和开发程序的简易性之间取得平衡。

一种架构的存储模型告诉了应用程序可以从它的存储系统中获得何种担保，同时详细定义了一些特殊的指令称为存储关卡（memory barriers）或栅栏（fences），用以在需要共享数据时，得到额外的存储协调保证。为了帮助java开发者屏蔽这些跨架构的存储模型之间的不同，java提供了自己的存储模型，JVM会通过在适当的位置上插入存储关卡，来解决JMM与底层平台存储模型之间的差异化。

跨线程共享数据时，现代可共享内存的多处理器（和编译器）架构会做出一些令人惊讶的事情来：除非你自己已经使用存储关卡，通知它们不要这么做。幸运的是，不需要在java程序中指明存储关卡的放置位置，只要在访问共享状态时能够识别到它们就可以了。通过正确地使用同步，可以做到这些。

PossibleReording执行可能得到的结果会有(0,1),(1,0),(1,1),(0,0)四种情况。其中第四种情况就是内存重排序的结果。因为两个线程没有依赖其他线程，线程中的赋值顺序可能会被重排。

```java
public class PossibleReording {
	static int x = 0, y = 0;
	static int a = 0, b = 0;

	public static void main(String[] args) throws InterruptedException {
		Thread one = new Thread(new Runnable() {
			@Override
			public void run() {
				a = 1;
				x = b;
			}
		});
		Thread other = new Thread(new Runnable() {
			@Override
			public void run() {
				b = 1;
				y = a;
			}
		});
		one.start();
		other.start();
		one.join();
		other.join();
		System.out.println("(" + x + "," + y + ")");
	}
}
```

java存储模型的定义是通过动作的形式描述的。所谓动作，包括变量的读和写、监视器加锁和释放锁、线程的启动和拼接。

JMM为所有程序内部的动作定义了一个偏序关系，叫做happens-before。

当一个变量被多个线程读取，且至少被一个线程写入时，如果读写操作并未依照happens-before排序，就会产生数据竞争。一个正确同步的程序是没有数据竞争的程序。

happens-before的法则包括：

- 程序次序法则：线程中的每个动作A都happens-before于该线程的每一个动作B，其中，在程序中，所有的动作B都出现在动作A之后。
- 监视器锁法则：对一个监视器锁的解锁happens-before于每一个后续对同一监视器锁的加锁。
- volatile变量法则：对volatile域的写入操作happens-before于每一个后续对同一域的读操作。
- 线程启动法则：在一个线程里，对Thread.start的调用会happens-before于每一个启动线程中的动作。
- 线程终结法则：线程中的任何动作都happens-before于其他线程检测到这个线程已经终结、或者从Thread.join调用中成功返回，或者Thread.isAlive返回false。
- 中断法则：一个线程调用另一个线程的interrupt happens-before于被中断的线程发现中断（通过抛出InterruptedException，或者调用isInterrupted和interrupted）。
- 终结法则：一个对象的构造函数的结束happens-before于这个对象finalizer的开始。
- 传递性：如果A happens-before于B，且B happens-before于C，则A happens-before于C。

为了满足happens-before，这需要结合“程序次序法则”与另外一种次序法则（通常是“监视器锁法则”或“volatile变量法则”）来对访问变量的操作进行排序，否则就用锁来保护它。

由类库担保的其他happens-before排序包括：

- 将一个条目置入线程安全容器happens-before于另一个线程从容器中获取条目。
- 执行CountDownLatch中的倒计时happens-before于线程从闭锁（latch）的await中返回。
- 释放一个许可给Semaphore happens-before于从同一Semaphore里获得一个许可。
- Future表现的任务所发生的动作happens-before于另一个线程成功地从Future.get中返回。
- 向Executor提交一个Runnable或Callable happens-before于开始执行任务。
- 最后，一个线程到达CyclicBarrier或Exchanger happens-before于相同关卡（barrier）或Exchange点中的其他线程被释放。如果CyclicBarrier使用一个关卡（barrier）动作，到达关卡happens-before于关卡动作，依照次序，关卡动作happens-before于线程从关卡中释放。

### 发布
一个没有安全发布的对象不能保证共享引用happens-before于另外的线程加载这个共享引用，那么写入新对象的引用与写入对象域可以被重新排序。在这种情况下，另一个线程可以看到对象引用的最新值，不过也看到一些或全部对象状态的过期值——一个部分创建对象。

我们常用的延迟加载就存在不安全发布的问题：

```java
public class UnsafeLazyInitialization {
	private static Resource resource;

	public static Resource getInstance() {
		if (resource == null) {
			resource = new Resource();
		}
		return resource;
	}
}
```

并发环境下，除了可能会重复生成实例外。还有可能线程A正在生成实例时，线程B也调用了getInstance，线程B看到resource已经有了非空值，于是就使用这个没有完全创建的resource。

除了不可变对象以外，使用被另一个线程初始化的对象，是不安全的，除非对象的发布是happens-before于对象的消费线程使用它。

如果getInstance不会被频繁访问，而且代码路径很简短，所以对这个方法加锁不会影响太大的性能：

```java
public class SafeLazyInitialization {
	private static Resource resource;

	public synchronized static Resource getInstance() {
		if (resource == null) {
			resource = new Resource();
		}
		return resource;
	}
}
```

如果使用主动初始化的技术，就更能够保证安全发布：

```java
public class EagerInitialization {
	private static Resource resource = new Resource();

	public static Resource getInstance() {
		return resource;
	}
}
```

静态初始化是由JVM完成的，发生在类的初始阶段，即类被其他线程调用之前。JVM要在初始化期间获得一个锁，这个锁每个线程都至少会用到一次，来确保一个类是否以北加载。这个锁也保证了静态初始化期间，内存写入的结果自动地对所有线程是可见的。所以静态初始化的对象，无论是构造期间还是被引用的时候，都不需要显式地进行同步。这仅仅适用于构造当时的状态，如果对象是可变的，为了保证后续修改的可变性，避免脏数据，读和写仍然需要同步。

JVM还有一项惰性类加载技术，即在使用类的时候才会加载类。我们可以利用这项技术实现惰性初始化。

```java
public class ResourceFactory {
	private static class ResourceHolder {
		public static Resource resource = new Resource();
	}

	public static Resource getResource() {
		return ResourceHolder.resource;
	}
}
```

私有的静态内部类在不使用的时候不会被JVM加载，在使用的时候就会被JVM加载的同时实例化静态作用域resource。

在JVM上古时代，初始化类需要消耗很大的资源，所以催生了双检查锁模式：

```java
public class DoubleCheckedLocking {
	private static Resource resource;

	public static Resource getInstance() {
		if (resource == null) {
			synchronized (DoubleCheckedLocking.class) {
				if (resource == null) {
					resource = new Resource();
				}
			}
		}
		return resource;
	}
}
```

实际上，这也同样存在之前的部分创建的问题。如果是部分创建，不会进入锁，也就没法保证可见性了。如果把resource改成volatile就可以保证可见性了。现代JVM效率已经不需要再使用双检查锁了。

### 初始化安全性
初始化安全可以保证，对于正确创建的对象，无论它是如何发布的，所有线程都将看到构造函数设置的final域的值。更进一步，一个正确创建的对象中，任何可以通过其final域触及到的变量（比如一个final数组中的元素，或者一个final域引用的HashMap里面的内容），也可以保证对其他线程都是可见的。

对于含有final域的对象，初始化安全可以抑制重排序。用于向通过final域可到达的初始化变量写入值的操作，不会和构造后的操作一起被重排序。

下面的代码即使存在着不安全的惰性初始化，或者在没有同步的公共静态域中隐藏SafeStates的引用，即使它没有使用同步，而且依赖于非线程安全的HashSet，它都可以被安全地发布。

```java
import java.util.HashMap;
import java.util.Map;

public class SafeStates {
	private final Map<String, String> states;

	public SafeStates() {
		states = new HashMap<String, String>();
		states.put("alaska", "AK");
		states.put("alabama", "AL");
		...
		states.put("wyoming", "WY");
	}

	public String getAbbreviation(String s) {
		return states.get(s);
	}
}
```

如果states不是final类型的，或者在构造函数以外的方法修改states的内容，初始化安全性就不足以在缺少同步的情况下保证安全地访问SafeStates。

初始化安全性保证只有以通过final域触及的值，在构造函数完成时才是可见的。对于通过非final域触及的值，或者创建完成后可能改变的值，必须使用同步来确保可见性。