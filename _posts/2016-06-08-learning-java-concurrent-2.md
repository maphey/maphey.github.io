---
date: 2016-06-11 20:49:12+08:00
layout: post
title: 一起学java并发编程（二）：线程的安全性
thread: 4
categories: java
tags: java并发编程
---
### 没有同步的共享状态域
<p>无共享状态域的多线程程序只适合一部分的利用场景，事实上很多多线程是为了合作完成一项任务，这些多线程程序需要线程间通信进行协作。一般情况下，线程间会使用共享状态域进行数据状态共享。这些共享的状态域就不再是线程安全的。</p>
<p>假如我们在之前计算因数的类需要添加一个统计计算因数的数字个数的功能，使用一个共享状态域，修改StatelessFactorizer.java如下所示：</p>
```java
import java.util.ArrayList;
import java.util.List;

public class StatelessFactorizer {
	// 包含的域导致了状态共享，不是线程安全的
	private int count = 0;

	public void service(Integer num) {
		List<Integer> factors = factor(num);
		count++;
		send(factors);
	}

	public int getCount() {
		return count;
	}

	private List<Integer> factor(int n) {
		List<Integer> res = new ArrayList<Integer>();
		int m = n;
		for (int i = 2; i <= n; i++) {
			if (fun(i) && m % i == 0) {
				res.add(i);
				m = m / i;
				i = 1;
				if (fun(m)) {
					break;
				}
			}
		}
		res.add(m);
		return res;
	}

	private boolean fun(int n) {
		for (int i = 2; i <= Math.sqrt(n); i++)
			if (n % i == 0)
				return false;
		return true;
	}

	private void send(List<Integer> factors) {
		System.out.println(Thread.currentThread().getName() + ": send fact:" + factors + "; total times:" + count);
	}
}
```
<p>我们编写一个Main.java类，来运行这段代码：</p>
```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		StatelessFactorizer statelessFactorizer = new StatelessFactorizer();
		Thread[] threads = new Thread[100];
		for (int i = 0; i < threads.length; i++) {
			Task task = new Task(statelessFactorizer, i);
			threads[i] = new Thread(task, "thread " + i);
			threads[i].start();
		}
		for (int i = 0; i < threads.length; i++) {
			threads[i].join();
		}
		System.out.println("total times: " + statelessFactorizer.getCount());
	}
}
```
<p>运行结果如下：</p>
>thread 903: send fact:[3, 7, 43]; total times:4  
>thread 914: send fact:[2, 457]; total times:6  
>thread 910: send fact:[2, 5, 7, 13]; total times:4  
>thread 906: send fact:[2, 3, 151]; total times:7  
>thread 912: send fact:[2, 2, 2, 2, 3, 19]; total times:4  
>...  
>thread 988: send fact:[2, 2, 13, 19]; total times:95  
>thread 997: send fact:[997, 1]; total times:95  
>thread 972: send fact:[2, 2, 3, 3, 3, 3, 3]; total times:96  
>thread 984: send fact:[2, 2, 2, 3, 41]; total times:97  
>thread 996: send fact:[2, 2, 3, 83]; total times:98  
>thread 992: send fact:[2, 2, 2, 2, 2, 31]; total times:99  
>total times: 99  

<p>最终结果是99（应该是100），结果并不正确。</p>
<p>有时我们想使用“先检查，再使用”的延迟初始化，在使用时才初始化对象，同时保证对象指挥被初始化一次，我们可能会使用下面这样的处理方式：</p>
```java
public class LazyInitRace {
	// 包含了域，导致了状态共享。
	private Object instance = null;

	public Object getInstance() {
		try {
			Thread.sleep(10);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		if (instance == null) {
			instance = new Object();
		}
		return instance;
	}
}
```
<p>这是典型的没有同步共享状态导致的线程不安全。</p>
```java
public class Main {
	public static void main(String[] args) {
		final LazyInitRace lazyInitRace = new LazyInitRace();
		for (int i = 0; i < 10; i++) {
			Thread thread = new Thread(new Runnable() {
				@Override
				public void run() {
					Object instance = lazyInitRace.getInstance();
					System.out.println(Thread.currentThread().getName() + ":" + instance);
				}
			}, "thread " + i);
			thread.start();
		}
	}
}
```
<p>运行结果如下：</p>
>thread 9:java.lang.Object@48258d58  
>thread 6:java.lang.Object@584aceca  
>thread 5:java.lang.Object@48258d58  
>thread 8:java.lang.Object@584aceca  
>thread 2:java.lang.Object@584aceca  
>thread 3:java.lang.Object@48258d58  
>thread 7:java.lang.Object@48258d58  
>thread 1:java.lang.Object@48258d58  
>thread 0:java.lang.Object@48258d58  
>thread 4:java.lang.Object@48258d58  

### 使用并发数据结构的共享状态域
<p>对于共享状态域，JDK1.5中提供了许多可供使用的并发数据类型，可以保证共享状态在多线程程序中线程安全型。</p>
```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class StatelessFactorizer {
	// 包含的域，但是该域是线程安全的，所以没有错误发生
	private final AtomicInteger count = new AtomicInteger(0);

	public void service(Integer num) {
		List<Integer> factors = factor(num);
		// 这个方法是原子的
		count.incrementAndGet();
		send(factors);
	}

	public int getCount() {
		return count.get();
	}

	private List<Integer> factor(int n) {
		List<Integer> res = new ArrayList<Integer>();
		int m = n;
		for (int i = 2; i <= n; i++) {
			if (fun(i) && m % i == 0) {
				res.add(i);
				m = m / i;
				i = 1;
				if (fun(m)) {
					break;
				}
			}
		}
		res.add(m);
		return res;
	}

	private boolean fun(int n) {
		for (int i = 2; i <= Math.sqrt(n); i++)
			if (n % i == 0)
				return false;
		return true;
	}

	private void send(List<Integer> factors) {
		System.out.println(Thread.currentThread().getName() + ": send fact:" + factors + "; total times:" + count.get());
	}
}
```
<p>这种情况下，由于这个共享状态域是线程安全的，所以不会出现共享变量可见性错误。能够保证线程安全性。在实际情况中，应尽可能的使用现有的线程安全对象来管理类的状态，因为线程安全对象的状态更容易维护和验证。</p>
<p>但是，并不是说使用了线程安全的并发类就是线程安全的。并发类与并发类进行数据交换的时候，也需要考虑状态同步的问题。假如我们有一个类UnSafeCachingFactorizer.java。用来记录最后一次计算的数字以及其因数：</p>
```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

// 非线程安全的
public class UnSafeCachingFactorizer {
	// 由于这里有两个原子引用，于是在原子引用和原子引用之间的修改就不是线程安全的
	private final AtomicReference<Integer> lastNumber = new AtomicReference<Integer>();
	private final AtomicReference<List<Integer>> lastFactors = new AtomicReference<List<Integer>>();

	public void service(Integer num) {
		if (num.equals(lastNumber.get())) {
			send(lastNumber.get());
		} else {
			List<Integer> factors = factor(num);
			lastNumber.set(num);
			// 上一步和这一步之间可能发生改变，不是线程安全的
			lastFactors.set(factors);
			send(factors);
		}
	}

	private void send(Object num) {
		System.out.println(Thread.currentThread().getName() + ":" + num);
	}

	private List<Integer> factor(int n) {
		List<Integer> res = new ArrayList<Integer>();
		int m = n;
		for (int i = 2; i <= n; i++) {
			if (fun(i) && m % i == 0) {
				res.add(i);
				m = m / i;
				i = 1;
				if (fun(m)) {
					break;
				}
			}
		}
		res.add(m);
		return res;
	}

	private boolean fun(int n) {
		for (int i = 2; i <= Math.sqrt(n); i++)
			if (n % i == 0)
				return false;
		return true;
	}
}
```
<p>尽管这两个共享状态域都是线程安全的，但是这两者之间的数据也是有状态的，在改变这两个共享状态域时，也需要保证数据状态同步。这里并没有同步数据状态，所以不是线程安全的。要保持状态的一致性，就需要在单个原子操作中更新所有相关的状态变量。</p>

### 使用锁的共享状态域
<p>为了保证数据状态同步，我们可以使用synchronized关键字保证线程同步。作为一种便利，每个对象都有一个内部锁，所以你不需要显式地创建锁对象。synchronized实质上是为同步的代码块加了一个内部锁，这个内部锁同时只能由一个线程获取，没有获取锁的线程必须等待已经获取锁的线程释放锁之后才能修改同步的共享状态。</p>
```java
import java.util.ArrayList;
import java.util.List;

// 对于每一个涉及多个变量的不变约束，需要同一个锁保护其所有变量。

/**
 * 所以形如Vector中这样的操作：
 * if(!vector.contains(element)){
 * 		vector.add(element);
 * }
 * 由于外面的方法可能没有同步（没有使用锁），所以外面的方法访问这两个方法并不能确保共享状态的一致性
 */
public class SynchronizedFactorizer {
	private Integer lastNumber;
	private List<Integer> lastFactors;

	public synchronized void service(Integer num) {
		if (num.equals(lastNumber)) {
			send(lastNumber);
		} else {
			List<Integer> factors = factor(num);
			lastNumber = num;
			lastFactors = factors;
			send(factors);
		}
	}

	private void send(Object num) {
		System.out.println(Thread.currentThread().getName() + ":" + num);
	}

	private List<Integer> factor(int n) {
		List<Integer> res = new ArrayList<Integer>();
		int m = n;
		for (int i = 2; i <= n; i++) {
			if (fun(i) && m % i == 0) {
				res.add(i);
				m = m / i;
				i = 1;
				if (fun(m)) {
					break;
				}
			}
		}
		res.add(m);
		return res;
	}

	private boolean fun(int n) {
		for (int i = 2; i <= Math.sqrt(n); i++)
			if (n % i == 0)
				return false;
		return true;
	}
}
```
<p>一个synchronized块有两部分：锁对象的引用，以及这个锁保护的代码块。synchronized方法是对跨越了这个方法体的synchronized块的简短描述，至于synchronized方法的锁，就是该方法所在的对象本身。（静态的synchronized方法从Class对象上获取锁）。</p>
<p>同时，锁具有“可重入”的特性，这样一个线程如果试图获取自己持有的锁，这个请求就会成功。重入的一种实现方式就是为每个锁关联一个计数值和所有者线程，每次重入就会使计数加1，释放就是减1，当计数为0时锁就会被释放。</p>
<p>当一个线程请求其他线程已经占有的锁时，请求线程将被阻塞。然而内部锁是可重进入的，因此线程在试图获得它自己占有的锁时，请求就会成功。重进入意味着锁的请求是基于“per-thread”，而不是基于“per-invocation”的。</p>
<p>如下代码是可以成功调用的，就是因为锁具有可重入的特性：</p>
<p>基类Widget.java</p>
```java
public class Widget {
	public synchronized void doSomething() {
		System.out.println(Thread.currentThread().getName() + ": superclass: do something.");
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```
<p>子类LoggingWidget.java</p>
```java
// 由于锁是可重入的，所以一个线程可以重复进入自己已经获得的锁。
public class LoggingWidget extends Widget {

	@Override
	public synchronized void doSomething() {
		System.out.println(Thread.currentThread().getName() + ": subclass: do something");
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		// 可重入锁，所以不会发生死锁
		super.doSomething();
	}
}
```
<p>但是，使用锁来保护一大段需要共享的状态导致性能十分低下，因为锁导致了大量代码的串行访问。所以应该尽量避免将不必要同步的变量放在同步代码块内。当然，请求与释放锁的操作是需要开销，所以将synchronized分解的过于琐碎是不合理的。</p>
```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * 这里没有使用并发变量，因为我们已经使用了synchronized构造块。使用两种不同的同步机制会引起混淆， 而且性能与安全也不能得到额外的好处
 * 请求与释放锁的操作是需要开销，所以将synchronized分解的过于琐碎是不合理的。
 * 决定synchronized块的大小需要权衡各种设计要求，包括安全性、简单性和性能。
 * 
 * 通常简单性与性能之间是相互牵制的。实现一个同步策略时，不要过早地为了性能而牺牲简单性。
 * 有些耗时的计算或操作，比如I/O，难以快速的完成，执行这些操作期间不要占有锁。
 */
public class CachedFactorizer {
	private Integer lastNumber;
	private List<Integer> lastFactors;
	private long hits;
	private long cacheHits;

	public long getHits() {
		return hits;
	}

	public double getCacheHits() {
		return (double) cacheHits / (double) hits;
	}

	public void service(Integer num) {
		List<Integer> factors = null;
		synchronized (this) {
			++hits;
			if (num.equals(lastNumber)) {
				++cacheHits;
				Collections.copy(factors, lastFactors);
			}
		}
		if (factors == null) {
			factors = factor(num);
			synchronized (this) {
				lastNumber = num;
				Collections.copy(lastFactors, factors);
			}
		}
		send(factors);
	}

	private void send(Object num) {
		System.out.println(Thread.currentThread().getName() + ":" + num);
	}

	private List<Integer> factor(int n) {
		List<Integer> res = new ArrayList<Integer>();
		int m = n;
		for (int i = 2; i <= n; i++) {
			if (fun(i) && m % i == 0) {
				res.add(i);
				m = m / i;
				i = 1;
				if (fun(m)) {
					break;
				}
			}
		}
		res.add(m);
		return res;
	}

	private boolean fun(int n) {
		for (int i = 2; i <= Math.sqrt(n); i++)
			if (n % i == 0)
				return false;
		return true;
	}
}
```
<p>所以，合适大小的代码块；同步代码的处理风格与易理解性；同步线程的使用场景，都影响着我们使用锁的方式。总之，<strong>所有并发问题都归结为如何协调访问并发状态，可变状态越少，保证线程安全就越容易。</strong></p>