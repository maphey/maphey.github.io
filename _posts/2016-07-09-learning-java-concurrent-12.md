---
date: 2016-07-09 22:16:12+08:00
layout: post
title: 一起学java并发编程（十二）：线程池的实践
thread: 14
categories: java
tags: java并发编程
---
单纯的使用“每任务每线程（thread-per-task）”方法存在一些缺陷，尤其是创建大量线程时更加突出。
- 线程生命周期开销。线程的创建与关闭需要在JVM和操作系统之间进行相应的处理活动。如果请求频繁且轻量，就会消耗大量的计算资源。
- 资源消耗量。活动线程会消耗系统资源，尤其是内存。运行的线程过多会导致线程切换频繁，导致CPU资源竞争。大量空闲线程占用更多内存，也给垃圾回收带来压力。
- 稳定性。受限于JVM和平台环境，过多的线程很可能超出资源限制，最可能就是收到一个OutOfMemoryError。

### Executors创建ThreadPoolExecutor
Executor提供了一个灵活的线程池实现。在Java类库中，任务执行的首要抽象不是Thread，而是Executor。Executor只是个简单的接口，但是它却为一个灵活而且强大的框架创造了基础。
和Java其他的类库类似Executors作为Executor的工具类，提供了许多简单实践。
假设我们创建100个线程，我们使用Executors创建Executor来执行这些任务。
创建执行的任务。

```java
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class Task implements Runnable {
	private Date initDate;
	private String name;

	public Task(String name) {
		initDate = new Date();
		this.name = name;
	}

	@Override
	public void run() {
		System.out.println(Thread.currentThread().getName() + ": task " + name + ":created on :" + initDate);
		System.out.println(Thread.currentThread().getName() + ": task " + name + ":started on :" + new Date());
		long duration = (long) Math.random() * 10;
		System.out.println(Thread.currentThread().getName() + ": task " + name + ":doing a task during :" + duration + " seconds");
		try {
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println(Thread.currentThread().getName() + ": task " + name + ":finished on:" + new Date());
	}
}
```

使用Executors创建线程池。

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

public class Server {
	private ThreadPoolExecutor executor;

	public Server() {
		executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();
	}

	public void executeTask(Task task) {
		System.out.println("server: a new task has arrived");
		executor.execute(task);
		System.out.println("server: pool size:" + executor.getPoolSize());
		System.out.println("server: active count:" + executor.getActiveCount());
		System.out.println("server: completed tasks:" + executor.getCompletedTaskCount());
	}

	public void endServer() {
		executor.shutdown();
	}
}
```

运行并查看结果。

```java
public class Main {
	public static void main(String[] args) {
		Server server = new Server();
		for (int i = 0; i < 100; i++) {
			Task task = new Task("task " + i);
			server.executeTask(task);
		}
		server.endServer();
	}
}
```

我们使用newCachedThreadPool方法创建线程池，这种线程池没有数量上的限制，只适合任务数量合理或者线程只会运行很短时间的线程。当线程池的长度超过了处理的需要时，它可以灵活地回收空闲的线程，当需求增加则可以添加新的线程。

无论何时当你看到这种形式的代码：
```java
new Thread(runnable).start()
```
并且你可能最终希望获得一个更加灵活的执行策略时，请认真考虑使用Executor代替Thread。

### Executors创建固定大小的线程池
Executors提供了有界队列防止应用程序过载而耗尽内存。通过创建固定大小的线程池可以避免由于大量任务导致的系统负载过高。
newFixedThreadPool将创建一个固定长度的线程池，每当提交一个任务就创建一个线程，直到达到线程池的最大数量，这时线程池的规模将不再变化（如果某个线程由于异常结束，线程池会自动补充一个新的线程）。
假设我们有容量为5的线程池。
```java
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

public class Server {
	private ThreadPoolExecutor executor;

	public Server() {
		executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(5);
	}

	public void executeTask(Task task) {
		System.out.println("server: a new task has arrived");
		executor.execute(task);
		System.out.println("server: pool size:" + executor.getPoolSize());
		System.out.println("server: active count:" + executor.getActiveCount());
		System.out.println("server: completed tasks:" + executor.getCompletedTaskCount());
		System.out.println("server: task count:" + executor.getTaskCount());
	}

	public void endServer() {
		executor.shutdown();
	}
}
```
```java
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class Task implements Runnable {
	private Date initDate;
	private String name;

	public Task(String name) {
		initDate = new Date();
		this.name = name;
	}

	@Override
	public void run() {
		System.out.println(Thread.currentThread().getName() + ": task " + name + ":created on :" + initDate);
		System.out.println(Thread.currentThread().getName() + ": task " + name + ":started on :" + new Date());
		long duration = (long) Math.random() * 10;
		System.out.println(Thread.currentThread().getName() + ": task " + name + ":doing a task during :" + duration + " seconds");
		try {
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println(Thread.currentThread().getName() + ": task " + name + ":finished on:" + new Date());
	}
}
```
```java
public class Main {
	public static void main(String[] args) {
		Server server = new Server();
		for (int i = 0; i < 100; i++) {
			Task task = new Task("task " + i);
			server.executeTask(task);
		}
		server.endServer();
	}
}
```

我们创建了一个可以生成5个线程的线程池，尽管有100个任务需要完成，但这100个任务只能由这5个线程完成，不会占用额外的系统资源。
我们还调用了shutdown()方法，关闭线程池。

### 获取ExecutorService执行任务的结果
默认的Thread类和Runnable接口没有返回执行结果。有时候我们却需要任务的执行结果。这时使用Executor框架可以实现这个功能。
Java并发API通过一下两个接口来实现这个功能。
Callable：这个接口声明了call()方法。可以在这个方法里实现任务的具体逻辑操作。Callable接口是一个泛型接口，这就意味着必须声明call()方法返回的数据类型。
Future：这个接口声明了一些方法来获取由Callable对象产生的结果，并管理它们的状态。
我们实现一个Callable类FactorialCalculator，假设进行一些耗时的计算。
```java
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

public class FactorialCalculator implements Callable<Integer> {
	private Integer number;

	public FactorialCalculator(Integer number) {
		this.number = number;
	}

	@Override
	public Integer call() throws Exception {
		int result = 1;
		if (number == 0 || number == 1) {
			result = 1;
		} else {
			for (int i = 0; i < number; i++) {
				result *= 1;
				TimeUnit.MILLISECONDS.sleep(20);
			}
		}
		System.out.println(Thread.currentThread().getName() + ": " + result);
		return result;
	}
}
```
我们使用submit方法将Callable对象发送给执行器执行，会返回Future对象。Future对象可以用于一下两个主要目的。
控制任务的状态：可以取消任务和检查任务是否已经完成。可使用isDone()方法检查任务是否已经完成。
通过call()方法获取返回的结果。为了达到这个目的，可以使用get()方法。这个方法一直等待直到Callable对象的call()方法执行完成并返回结果。如果get()方法在等待结果时线程中断了，则将抛出一个InterruptedException异常。
```java
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) {
		ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(2);
		List<Future<Integer>> resultList = new ArrayList<Future<Integer>>();
		Random random = new Random();
		for (int i = 0; i < 10; i++) {
			Integer number = random.nextInt(10);
			FactorialCalculator calculator = new FactorialCalculator(number);
			Future<Integer> result = executor.submit(calculator);
			resultList.add(result);
		}
		do {
			System.out.println("main: number of completed tasks:" + executor.getCompletedTaskCount());
			for (int i = 0; i < resultList.size(); i++) {
				Future<Integer> result = resultList.get(i);
				System.out.println("main: task " + i + ":" + result.isDone());
			}
			try {
				TimeUnit.MILLISECONDS.sleep(50);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		} while (executor.getCompletedTaskCount() < resultList.size());
		System.out.println("main: result");
		for (int i = 0; i < resultList.size(); i++) {
			Future<Integer> result = resultList.get(i);
			Integer number = null;
			try {
				number = result.get();
			} catch (InterruptedException | ExecutionException e) {
				e.printStackTrace();
			}
			System.out.println("main: task " + i + ":" + number);
		}
		executor.shutdown();
	}
}
```

Future还提供了一个超时的get方法，当获取任务结果一直阻塞到超时时间时就会返回null。

### 使用ExecutorService返回第一个成功的结果

### 使用ExecutorService返回所有结果

### 使用ScheduledThreadPoolExecutor执行延时任务

### 使用ScheduledThreadPoolExecutor执行周期任务

### 取消执行器中的任务
Executors的子接口ExecutorService暗示了生命周期有3中状态：运行、关闭和终止。ExecutorService最初创建后的初始状态是运行状态。shutdown方法会启动一个平缓的关闭过程：停止接收新的任务，同时等待已经提交的任务完成——包括尚未开始执行的任务。shutdownNow方法会启动一个强制关闭过程：尝试取消所有运行中的任务和排在队列中尚未开始的任务。

### 添加执行器FutureTask完成后的后续处理

### CompletionService分离任务启动与结果处理

### 处理被执行器拒绝的任务

### 任务取消