---
date: 2016-06-28 22:16:12+08:00
layout: post
title: 一起学java并发编程（九）：线程实践
thread: 11
categories: java
tags: java并发编程
---
### 线程的中断
在java API和语言规范中，并没有把中断与任何取消的语意绑定起来，但是，实际上，使用中断来处理取消之外的任何事情都是不明智的，并且很难支持起更大的应用。

静态的interrupted方法名并不理想，它仅仅能够清除当前线程的中断状态，并返回它之前的值：这是清除中断状态唯一的方法。

阻塞库函数，比如：Thread.sleep和Object.wait，试图检测线程何时被中断，并提前返回。它们对中断响应表现为：清除中断状态，抛出InterruptException：这表示阻塞操作因为中断的缘故提前结束。JVM并没有对阻塞方法发现中断的速度作出保证，不过在现实中这样的响应还是比较迅速的。

调用interrupt并不意味着必然停止目标线程正在进行的工作；它仅仅传递了请求中断的消息。

静态的interrupted应该小心使用，因为它会清除并发线程的中断状态。如果你调用了interrupted，并且它返回了true，你必须对其进行处理，除非你想掩盖这个中断——你可以抛出InterruptedException，或者通过再次调用interrupt来保存中断状态。

下面是通过中断进行取消的例子：

PrimeGenerator.java定义了一个可以取消的任务：
```java
public class PrimeGenerator extends Thread {

	@Override
	public void run() {
		long number = 1l;
		while (true) {
			if (isPrime(number)) {
				System.out.printf("Number %d is prime\n", number);
			}
			if (isInterrupted()) {
				System.out.println("the prime generator has been interrupted");
				return;
			}
			number++;
		}
	}

	private boolean isPrime(long number) {
		if (number < 2) {
			return true;
		}
		for (int i = 2; i < number; i++) {
			if (number % i == 0) {
				return false;
			}
		}
		return true;
	}
}
```
Main.java调用interrupted触发取消：
```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		Thread task = new PrimeGenerator();
		task.start();
		Thread.sleep(1000);
		task.interrupt();
	}
}
```
上面的例子中，Main调用了interrupt方法，会设置并保存task线程的取消状态。

中断异常通常是实现取消最明智的选择。正如需要为任务制定取消策略一样，也应该制定线程中断策略。一个中断策略决定线程如何应对中断请求——当发现中断请求时，它会做什么，哪些工作单元对于中断来说是原子操作，以及在多快的时间里响应中断。

当我们调用可中断的阻塞方法时，通常并不能得到期望的响应，对中断状态进行显式的检测会对此有一定帮助。

中断策略中最有意义的是对线程级和服务级取消的规定：尽可能迅速退出，如果需要的话进行清理，可能的话通知其拥有的实体，这个线程已经退出。很可能建立其他中断策略，比如暂停和重新开始，但是那些具有非标准中断策略的线程或线程池，需要被约束于那些应用了该策略的任务中。

一个任务不应该假设其执行线程的中断策略，除非它显式地设计用来运行在服务中，并且这些服务有明确的中断策略。无论任务把中断解释为取消，还是其他的一些关于中断的操作，它都应该注意保存执行线程的中断状态。如果对中断的处理不仅仅是把 InterruptException 传递给调用者，那么它应该在捕获 InterruptException 之后恢复中断的状态：

```java
	Thread.currentThread().interrupt();
```

或

```java
	interrupt();
```

因为每一个线程都有其自己的中断策略，所有你不应该中断线程，除非你知道中断对这个线程意味着什么。

有两种处理 InterruptException 的使用策略：
- 传递异常，将异常往上级抛；
- 保存中断状态，上层调用栈中的代码能够对其进行处理。

只有实现了线程中断策略的代码才可以接收中断请求。通用目的的任务和库的代码绝不应该接收中断请求。

FileSearch.java启动一个线程搜索一个文件：

```java
import java.io.File;

public class FileSearch implements Runnable {
	private String initPath;
	private String fileName;

	public FileSearch(String initPath, String fileName) {
		this.initPath = initPath;
		this.fileName = fileName;
	}

	@Override
	public void run() {
		File file = new File(initPath);
		if (file.isDirectory()) {
			boolean interrupted = false;
			try {
				directoryProcess(file);
			} catch (InterruptedException e) {
				interrupted = true;
				System.out.println(Thread.currentThread().getName() + ": the search has been interrupted");
			} finally {
				if (interrupted) {
					// 这里线程发生了中断，保存中断信息，保证上层的代码可以进行进一步处理
					Thread.currentThread().interrupt();
				}
			}
		}
	}

	private void directoryProcess(File file) throws InterruptedException {
		File[] list = file.listFiles();
		if (list != null) {
			for (int i = 0; i < list.length; i++) {
				if (list[i].isDirectory()) {
					directoryProcess(list[i]);
				} else {
					fileProcess(list[i]);
				}
			}
		}
		if (Thread.interrupted()) {
			throw new InterruptedException();
		}
	}

	private void fileProcess(File file) throws InterruptedException {
		if (file.getName().equals(fileName)) {
			System.out.println(Thread.currentThread().getName() + " : " + file.getAbsolutePath());
		}
		if (Thread.interrupted()) {
			throw new InterruptedException();
		}
	}
}
```
Main.java启动这个线程，如果10秒没有找到就取消任务：
```java
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) throws InterruptedException {
		FileSearch searcher = new FileSearch("c:\\", "img13.jpg");
		Thread thread = new Thread(searcher);
		thread.start();
		TimeUnit.SECONDS.sleep(10);
		thread.interrupt();
	}
}
```

ExecutorService.Submit会返回一个Future来描述任务。Future有一个cancel方法，它需要一个boolean类型的参数mayInterruptIfRunning，它的返回值表示取消尝试是否成功。当mayInterruptIfRunning为true，并且任务当前正在运行于一些线程中，那么这个线程是应该中断的。把这个参数设置成false意味着“如果还没有启动的话，不要运行这个任务”，这应该用于那些不处理中断的任务。

并不是所有的阻塞方法或阻塞机制都响应中断。对于那些被不可中断的活动所阻塞的线程，我们可以使用与中断类似的手段，来确保停止这些线程。
- java.io中的同步Socket I/O。在服务器应用程序中，阻塞I/O最常见的形式是读取和写入Socket。不幸的是，InputStream和OutputStream中的read和write方法都不响应中断，但是通过关闭底层的Socket，可以让read和write所阻塞的线程抛出一个SocketException。
- java.nio中的同步I/O。中断一个InterruptibleChannel的线程，会导致抛出CLosedByInterruptException，并关闭链路（也会导致其他线程在这条链路的阻塞，抛出CloseByInterruptException）。关闭一个InterruptibleException导致多个阻塞在链路操作上的线程抛出AsynchronousCloseException。
- Selector的异步I/O。如果一个线程阻塞于Selector.select方法，close方法会导致它通过抛出ClosedSelectorException提前返回。
- 获得锁。如果一个线程在等待内部锁，那么如果不能确保它最终获得锁，并且作出足够多的努力，让你能够以其他方式获得它的注意，你是不能停止它的。然而，显示Lock类提供了lockInterruptible方法，允许你等待一个锁，并仍能够响应中断。

我们可以通过覆写Thread的interrupt方法实现这种不可中断的阻塞：

```java
import java.io.IOException;
import java.io.InputStream;
import java.net.Socket;

public class ReaderThread extends Thread {
	private final Socket socket;
	private final InputStream in;

	public ReaderThread(Socket socket) throws IOException {
		this.socket = socket;
		this.in = socket.getInputStream();
	}

	@Override
	public void run() {
		byte[] buf = new byte[BUFSZ];
		while (true) {
			int count = in.read(buf);
			if (count < 0) {
				break;
			} else if (count > 0) {
				processBuffer(buf, count);
			}
		}
	}

	@Override
	public void interrupt() {
		try {
			socket.close();
		} catch (IOException e) {
		} finally {
			super.interrupt();
		}
	}
}
```

执行器有自己的关闭策略，将在执行器中介绍。

### sleep
sleep方法为线程指定了一段休眠时间，在这段时间内，线程不占用任何资源，TimeUnit的枚举元素也是使用Thread类的sleep方法实现的。

```java
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class FileClock implements Runnable {

	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			System.out.println(new Date());
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				System.out.println("the fileclock has been interrupted");
			}
		}
	}
}
```
```java
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) throws InterruptedException {
		FileClock clock = new FileClock();
		Thread thread = new Thread(clock);
		thread.start();
		TimeUnit.SECONDS.sleep(5);
		thread.interrupt();
	}
}
```

### join
join可以用来等待线程终止。当一个线程对象的join方法被调用时，调用它的线程将被挂起，直到这个线程对象完成它的任务。

假设我们有两个初始化资源的线程，必须等待资源初始化完成后，才能继续执行。

```java
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class DataSourceLoader implements Runnable {

	@Override
	public void run() {
		System.out.println("beginging data sources loading:" + new Date());
		try {
			TimeUnit.SECONDS.sleep(4);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("data sources loading has finished:" + new Date());
	}
}
```
```java
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class NetworkConnectionsLoader implements Runnable {

	@Override
	public void run() {
		System.out.println("beginging network connection loading:" + new Date());
		try {
			TimeUnit.SECONDS.sleep(6);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("data network connection loading has finished:" + new Date());
	}
}
```
```java
import java.util.Date;

public class Main {
	public static void main(String[] args) throws InterruptedException {
		DataSourceLoader dsLoader = new DataSourceLoader();
		Thread thread1 = new Thread(dsLoader);
		NetworkConnectionsLoader networkConnectionLoader = new NetworkConnectionsLoader();
		Thread thread2 = new Thread(networkConnectionLoader);
		thread1.start();
		thread2.start();
		thread1.join();
		thread2.join();
		System.out.println("Main: configuration has been loaded:" + new Date());
	}
}
```

### 守护线程
java里有一种特殊的线程，叫守护线程。这种线程的优先级很低，当同一个应用程序里没有其他线程运行的时候，守护线程才允许。当守护线程是程序唯一运行的线程时，守护线程执行结束后，JVM也就结束了这个程序。

```java
import java.util.Date;
import java.util.Deque;

public class CleanerTask extends Thread {
	private Deque<Event> deque;

	public CleanerTask(Deque<Event> deque) {
		this.deque = deque;
		setDaemon(true);
	}

	@Override
	public void run() {
		while (true) {
			Date date = new Date();
			clean(date);
		}
	}

	private void clean(Date date) {
		long difference;
		boolean delete;
		if (deque.size() == 0) {
			return;
		}
		delete = false;
		do {
			Event e = deque.getLast();
			difference = date.getTime() - e.getDate().getTime();
			if (difference > 10000) {
				System.out.println("cleaner: " + e.getEvent());
				deque.removeLast();
				delete = true;
			}
		} while (difference > 10000);
		if (delete) {
			System.out.println("cleaner: size of the queue:" + deque.size());
		}
	}
}
```
```java
import java.util.Date;

public class Event {
	private Date date;
	private String event;
	public Date getDate() {
		return date;
	}
	public void setDate(Date date) {
		this.date = date;
	}
	public String getEvent() {
		return event;
	}
	public void setEvent(String event) {
		this.event = event;
	}
}
```
```java
import java.util.Date;
import java.util.Deque;
import java.util.concurrent.TimeUnit;

public class WriterTask implements Runnable {
	private Deque<Event> deque;

	public WriterTask(Deque<Event> deque) {
		this.deque = deque;
	}

	@Override
	public void run() {
		for (int i = 0; i < 100; i++) {
			Event event = new Event();
			event.setDate(new Date());
			event.setEvent(String.format("the thread %s has generated an event", Thread.currentThread().getId()));
			deque.addFirst(event);
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```
```java
import java.util.ArrayDeque;
import java.util.Deque;

public class Main {
	public static void main(String[] args) {
		Deque<Event> deque = new ArrayDeque<Event>();
		WriterTask writerTask = new WriterTask(deque);
		for (int i = 0; i < 3; i++) {
			Thread thread = new Thread(writerTask);
			thread.start();
		}
		CleanerTask cleaner = new CleanerTask(deque);
		cleaner.start();
	}
}
```

### 线程中未处理异常的处理
run方法不支持throws语句，所以当线程对象的run方法抛出非运行时异常时，我们必须捕获并处理它们。

Java为我们提供了一种在线程对象里捕获和处理运行时异常的一种机制。

```java
public class Task implements Runnable {

	@Override
	public void run() {
		Integer.parseInt("TTT");
	}
}
```
```java
import java.lang.Thread.UncaughtExceptionHandler;

public class ExceptionHandler implements UncaughtExceptionHandler {

	@Override
	public void uncaughtException(Thread t, Throwable e) {
		System.out.println("An exception has been captured");
		System.out.println("thread: " + t.getId());
		System.out.println("exception: " + e.getClass().getName() + ":" + e.getMessage());
		System.out.println("stack trace:");
		e.printStackTrace();
		System.out.println("thread status:" + t.getState());
	}
}
```
```java
public class Main {
	public static void main(String[] args) {
		Task task = new Task();
		Thread thread = new Thread(task);
		thread.setUncaughtExceptionHandler(new ExceptionHandler());
		thread.start();
	}
}
```

### 线程组
Java能够对线程进行分组。这允许我们把一个组的线程当成一个单一的单元，对组内线程对象进行访问并操作它们。

```java
import java.util.Date;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class SearchTask implements Runnable {
	private Result result;

	public SearchTask(Result result) {
		this.result = result;
	}

	@Override
	public void run() {
		String name = Thread.currentThread().getName();
		System.out.println("thread " + name + ": start");
		try {
			doTask();
			result.setName(name);
		} catch (InterruptedException e) {
			System.out.println("thread " + name + ": interrupted");
			return;
		}
		System.out.println("thread " + name + ":END");
	}

	private void doTask() throws InterruptedException {
		Random random = new Random(new Date().getTime());
		int value = (int) (random.nextDouble() * 100);
		System.out.println("thread " + Thread.currentThread().getName() + ":" + value);
		TimeUnit.SECONDS.sleep(value);
	}
}
```
```java
public class Result {
	private String name;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}
}
```
```java
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) throws InterruptedException {
		ThreadGroup threadGroup = new ThreadGroup("searcher");
		Result result = new Result();
		SearchTask searchTask = new SearchTask(result);
		for (int i = 0; i < 5; i++) {
			Thread thread = new Thread(threadGroup, searchTask);
			thread.start();
			TimeUnit.SECONDS.sleep(1);
		}
		System.out.println("number if threads :" + threadGroup.activeCount());
		System.out.println("information about the thread group");
		threadGroup.list();
		Thread[] threads = new Thread[threadGroup.activeCount()];
		threadGroup.enumerate(threads);
		for (int i = 0; i < threads.length; i++) {
			System.out.println("thread " + threads[i].getName() + ":" + threads[i].getState());
		}
		waitFinish(threadGroup);
		threadGroup.interrupt();
	}

	private static void waitFinish(ThreadGroup threadGroup) throws InterruptedException {
		while (threadGroup.activeCount() > 9) {
			TimeUnit.SECONDS.sleep(1);
		}
	}
}
```

### 线程组中未处理异常的处理
与线程类似，线程组也提供了异常处理机制。

```java
public class MyThreadGroup extends ThreadGroup {

	public MyThreadGroup(String name) {
		super(name);
	}

	@Override
	public void uncaughtException(Thread t, Throwable e) {
		System.out.println("the thread " + t.getId() + " has thrown an Exception");
		e.printStackTrace();
		System.out.println("Terminating the rest of the Threads");
		interrupt();
	}
}
```
```java
import java.util.Random;

public class Task implements Runnable {

	@Override
	public void run() {
		int result;
		Random random = new Random(Thread.currentThread().getId());
		while (true) {
			result = 1000 / (random.nextInt(1000));
			System.out.println(Thread.currentThread().getId() + "" + result);
			if (Thread.currentThread().isInterrupted()) {
				System.out.println(Thread.currentThread().getId() + ":Interrupted");
				return;
			}
		}
	}
}
```
```java
public class Main {
	public static void main(String[] args) {
		MyThreadGroup threadGroup = new MyThreadGroup("mythreadgroup");
		Task task = new Task();
		for (int i = 0; i < 2; i++) {
			Thread t = new Thread(threadGroup, task);
			t.start();
		}
	}
}
```

### 自定义线程工厂类
java提供了ThreadFactory接口，可是实现线程对象工厂。使用线程对象工厂可以为我们定制、统计线程对象提供方便。

比如我们定制生成的线程名。

```java
import java.util.ArrayList;
import java.util.Date;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.ThreadFactory;

public class MyThreadFactory implements ThreadFactory {
	private int counter;
	private String name;
	private List<String> stats;

	public MyThreadFactory(String name) {
		counter = 0;
		this.name = name;
		stats = new ArrayList<String>();
	}

	@Override
	public Thread newThread(Runnable r) {
		Thread t = new Thread(r, name + "-Thread_" + counter);
		counter++;
		stats.add(String.format("create thread %d with name %s on %s", t.getId(), t.getName(), new Date()));
		return t;
	}

	public String getStats() {
		StringBuffer buffer = new StringBuffer();
		Iterator<String> it = stats.iterator();
		while (it.hasNext()) {
			buffer.append(it.next());
			buffer.append("\n");
		}
		return buffer.toString();
	}
}
```
```java
import java.util.concurrent.TimeUnit;

public class Task implements Runnable {

	@Override
	public void run() {
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

}
```
```java
public class Main {
	public static void main(String[] args) {
		MyThreadFactory factory = new MyThreadFactory("myThreadFactory");
		Task task = new Task();
		Thread thread;
		System.out.println("starting the threads");
		for (int i = 0; i < 10; i++) {
			thread = factory.newThread(task);
			thread.start();
		}
		System.out.println("factory stats:" + factory.getStats());
	}
}
```