---
date: 2016-06-03 11:49:12+08:00
layout: post
title: 一起学java并发编程（一）：并发编程最简单的方式
thread: 3
categories: java
tags: java并发编程
---
## 了解并发编程
<p>随着CPU处理能力和核心数的增长，为了尽力榨取CPU的性能，提高计算机的资源利用率。现代计算机操作系统都实现了多个程序并行执行的能力。并发编程有两种实现方式：多进程和多线程。多进程和多线程实现并发各有优劣（如下表），多线程在执行效率上具有明显的优势。在大多数现代操作系统中，都是以线程为基本的调度单位，而不是进程。因此，我们通常利用多线程的方式编写java并发程序。</p>
<table>
	<thead>
		<tr>
			<th>对比维度</th>
			<th>多进程</th>
			<th>多线程</th>
			<th>总结</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>数据共享、同步</td>
			<td>数据共享复杂，需要用IPC；数据是分开的，同步简单</td>
			<td>因为共享进程数据，数据共享简单，但也是因为这个原因导致同步复杂</td>
			<td>各有优势</td>
		</tr>
		<tr>
			<td>内存、CPU</td>
			<td>占用内存多，切换复杂，CPU利用率低</td>
			<td>占用内存少，切换简单，CPU利用率高</td>
			<td>线程占优</td>
		</tr>
		<tr>
			<td>创建销毁、切换</td>
			<td>创建销毁、切换复杂，速度慢</td>
			<td>创建销毁、切换简单，速度很快</td>
			<td>线程占优</td>
		</tr>
		<tr>
			<td>编程、调试</td>
			<td>编程简单，调试简单</td>
			<td>编程复杂，调试复杂</td>
			<td>进程占优</td>
		</tr>
		<tr>
			<td>可靠性</td>
			<td>进程间不会互相影响</td>
			<td>一个线程挂掉将导致整个进程挂掉</td>
			<td>进程占优</td>
		</tr>
		<tr>
			<td>分布式</td>
			<td>适应于多核、多机分布式；如果一台机器不够，扩展到多台机器比较简单</td>
			<td>适应于多核分布式</td>
			<td>进程占优</td>
		</tr>
	</tbody>
</table>
<p>由于现在计算机大都是多核甚至是多CPU，每个CPU核心都有一级甚至多级缓存。对象共享和状态成为多线程编程最大的难题。为了解决这个问题，有三种解决方法：</p>
- 不在线程之间共享该状态变量
- 将状态变量修改为不可变的变量
- 在访问状态变量时使用同步

<p>其中最简单的就是“不在线程之间共享状态变量”。因为不涉及到共享对象，所以不需要额外的操作来同步共享对象的状态。本节我们就探讨一下“无共享状态变量的并发”。</p>

## 了解java中的多线程
<p>在java中创建线程有两种方式：一种是继承Thread类，覆写run方法；另一种是实现Runnable接口，实现run方法，并将实现类传给Thread实例执行。两种方式的效果是一样的，只不过实现Runnable接口时可以另外再继承一个类，可用于某些需要复用基类的情况。</p>
<p>我们来看看使用Runnable实现多线程的一个例子。</p>
<p>Calculator.java，执行的线程：</p>
```java
public class Calculator implements Runnable {
	private int number;

	public Calculator(int number) {
		this.number = number;
	}

	@Override
	public void run() {
		for (int i = 1; i <= 10; i++) {
			System.out.println(Thread.currentThread().getName() + ":" + number + " * " + i + " = " + i * number);
		}
	}

}
```
<p>Main.java，用于创建、执行、获取线程的状态：</p>
```java
import java.lang.Thread.State;

public class Main {
	public static void main(String[] args) {
		Thread[] threads = new Thread[10];
		Thread.State status[] = new Thread.State[10];
		for (int i = 0; i < 10; i++) {
			Calculator calculator = new Calculator(i);
			threads[i] = new Thread(calculator);
			if (i % 2 == 0) {
				threads[i].setPriority(Thread.MAX_PRIORITY);
			} else {
				threads[i].setPriority(Thread.MIN_PRIORITY);
			}
			threads[i].setName("Thread new Name" + i);
		}
		for (int i = 0; i < 10; i++) {
			System.out.println("Main: status of thread " + i + " : " + threads[i].getState());
			status[i] = threads[i].getState();
		}
		for (int i = 0; i < 10; i++) {
			threads[i].start();
		}
		boolean finish = false;
		while (!finish) {
			for (int i = 0; i < 10; i++) {
				if (!threads[i].getState().equals(status[i])) {
					printThreadInfo(threads[i], status[i]);
					status[i] = threads[i].getState();
				}
			}
			finish = true;
			for (int i = 0; i < 10; i++) {
				finish = finish && (threads[i].getState().equals(State.TERMINATED));
			}
		}
	}

	private static void printThreadInfo(Thread thread, State state) {
		System.out.println("main: id " + thread.getId() + " - " + thread.getName());
		System.out.println("main: priority " + thread.getPriority());
		System.out.println("main: old state: " + state);
		System.out.println("main: new state: " + thread.getState());
		System.out.println("************");
	}
}
```
<p>Thread类有一些保存信息的属性，这些属性可以获取/设置线程的状态。</p>
- ID：保存线程唯一标识符
- Name：保存线程名称
- Priority：保存或设置线程的优先级。线程的优先级从1到10，其中1优先级最低，10最高。优先级会映射到操作系统的线程优先级，比如Windows一共有7个线程优先级。设置了优先级并不一定完全保证高优先级的线程一定会先执行，只是一个推荐作用，使高优先级的线程有更大的概率优先执行
- Status：保存线程的状态。java线程一个有6种状态：new、runnable、blocked、waiting、time waiting和terminated。

## 不共享状态变量的线程
<p>在java的线程中创建的变量被封装在线程内部，不需要也无法共享状态。这样的多线成程序无需同步共享状态变量。我们来看一个例子。</p>
<p>Calculator.java，实现了Runnable接口，并在类内部创建了一个List对象。这个List对象不是共享状态变量，无需同步就能保证线程安全：</p>
```java
class Calculator implements Runnable {
	private int number;

	public Calculator(int number) {
		this.number = number;
	}

	@Override
	public void run() {
		List<Integer> privateList = new ArrayList<Integer>();
		for (int i = 0; i < number; i++) {
			privateList.add(i);
		}
		for (int i = 0; i < 10; i++) {
			System.out.println(Thread.currentThread().getName() + ": create private variable :" + privateList);
		}
	}
}
```
<p>Main.java，创建多线程程序：</p>
```java
public class Main {
	public static void main(String[] args) {
		for (int i = 0; i < 10; i++) {
			Calculator calculator = new Calculator(i);
			Thread thread = new Thread(calculator);
			thread.start();
		}
	}
}
```

## 使用ThreadLocal避免共享状态变量
<p>ThreadLocal提供了另一种更规范方式避免了共享状态变量。ThreadLocal为每一个线程保存一份共享变量的副本，内部使用一个Map来维护共享变量，Map中元素的key就是线程对象，value为对应线程的变量副本。如此，每个线程都有自己的变量副本，也就没必要对共享变量进行同步了，从而隔离了多个线程对数据的访问冲突。我们来看一个例子。</p>
<p>UnsafeTask.java，创建一个Runnable实例，我们先看看如果不使用ThreadLocal，会怎么样：</p>
```java
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class UnsafeTask implements Runnable {
	private Date startDate;

	@Override
	public void run() {
		startDate = new Date();
		System.out.println("starting thread: " + Thread.currentThread().getId() + ":" + startDate);
		try {
			TimeUnit.SECONDS.sleep((int) Math.rint(Math.random() * 10));
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("thread finished:" + Thread.currentThread().getId() + ":" + startDate);
	}

}
```
<p>Core.java，创建这个Runnable实例，并在10个线程中执行之：</p>
```java
public class Core {
	public static void main(String[] args) throws InterruptedException {
		UnsafeTask task = new UnsafeTask();
		for (int i = 0; i < 10; i++) {
			Thread thread = new Thread(task);
			thread.start();
			TimeUnit.SECONDS.sleep(2);
		}
	}
}
```
<p>我们会发现，我们设定的每个线程的开始时间都不一样，但是当它们结束时却有一部分的线程打印的startDate值却是一样的。这就是共享状态变量没有同步造成的。</p>
<p>我们可以使用ThreadLocal来避免共享startDate变量。我们创建一个新的java类，SafeTask.java：</p>
```java
public class SafeTask implements Runnable {
	private ThreadLocal<Date> startDate = new ThreadLocal<Date>() {
		protected Date initialValue() {
			return new Date();
		}
	};

	@Override
	public void run() {
		System.out.println("starting thread: " + Thread.currentThread().getId() + ":" + startDate.get());
		try {
			TimeUnit.SECONDS.sleep((int) Math.rint(Math.random() * 10));
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("thread finished:" + Thread.currentThread().getId() + ":" + startDate.get());
	}

}
```
<p>同时，将Core.java修改如下：</p>
```java
public class Core {
	public static void main(String[] args) throws InterruptedException {
		SafeTask task = new SafeTask();
		for (int i = 0; i < 10; i++) {
			Thread thread = new Thread(task);
			thread.start();
			TimeUnit.SECONDS.sleep(2);
		}
	}
}
```
<p>使用了ThreadLocal之后，我们在每个线程中都保存了一份startDate的副本，彼此隔离，也就没有了共享变量同步的问题。</p>

### 小结
<p>并发编程最困难的地方在于共享变量状态的同步，尽量避免线程间共享变量能够极大的简化并发程序的编写。在java开发中，数据库连接池就是使用ThreadLocal的典型例子，因为数据库连接的数据是一样的，所以在每一个线程中保存一份连接池的副本能够避免连接数据的同步问题，同时能够使多线程程序很好的支持并发访问数据库。<br>
尽管这种没有共享变量的并发简单且有效，但是并不适合所有的场景，而且某些方法也有缺点。我们会在以后的章节中详细说明。
</p>