---
date: 2016-06-30 22:16:12+08:00
layout: post
title: 一起学java并发编程（十）：锁的实践
thread: 12
categories: java
tags: java并发编程
---
### synchronized的使用
synchronized是java内置锁，又称为重量级锁。synchronized默认的获取的锁是内置锁，对于同步方法就是当前实例对象，对于静态同步方法就是当前的Class对象。

假设有一个银行账户，公司往账户存钱，银行从账户扣钱。两者可以并发进行，这就需要对账户进行同步。

```java
public class Account {
	private double balance;

	public double getBalance() {
		return balance;
	}

	public void setBalance(double balance) {
		this.balance = balance;
	}

	public synchronized void addAmount(double amount) throws InterruptedException {
		double tmp = balance;
		Thread.sleep(10);
		tmp += amount;
		balance = tmp;
	}

	public synchronized void subtractAmount(double amount) throws InterruptedException {
		double tmp = balance;
		Thread.sleep(10);
		tmp -= amount;
		balance = tmp;
	}
}
```

```java
public class Company implements Runnable {
	private Account account;

	public Company(Account account) {
		this.account = account;
	}

	@Override
	public void run() {
		for (int i = 0; i < 100; i++) {
			try {
				account.addAmount(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

```java
public class Bank implements Runnable {
	private Account account;

	public Bank(Account account) {
		this.account = account;
	}

	@Override
	public void run() {
		for (int i = 0; i < 100; i++) {
			try {
				account.subtractAmount(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		Account account = new Account();
		account.setBalance(1000);
		Company company = new Company(account);
		Thread companyThread = new Thread(company);
		Bank bank = new Bank(account);
		Thread bankThread = new Thread(bank);
		System.out.println("account: initial balance:" + account.getBalance());
		companyThread.start();
		bankThread.start();
		companyThread.join();
		bankThread.join();
		System.out.println("account: final balance:" + account.getBalance());
	}
}
```

这里的Synchronized使用的就是内置锁，及账户实例。因为两者保护的是同一个账户，所以是同一个锁。

### 使用对象的synchronized
对于同步代码块就是Synchronized括号里配置的对象，同一个对象只能同时被一个线程访问。

假设我们有两个售票处，每个地方都可以售票和退票，假设每个售票处可以同时多人售退票。

```java
public class Cinema {
	private long vacanciesCinema1;
	private long vacanciesCinema2;
	private final Object controlCinema1;
	private final Object controlCinema2;

	public Cinema() {
		controlCinema1 = new Object();
		controlCinema2 = new Object();
		vacanciesCinema1 = 20;
		vacanciesCinema2 = 20;
	}

	public boolean sellTickets1(int number) {
		synchronized (controlCinema1) {
			if (number < vacanciesCinema1) {
				vacanciesCinema1 -= number;
				return true;
			} else {
				return false;
			}
		}
	}

	public boolean sellTickets2(int number) {
		synchronized (controlCinema2) {
			if (number < vacanciesCinema2) {
				vacanciesCinema2 -= number;
				return true;
			} else {
				return false;
			}
		}
	}

	public boolean returnTickets1(int number) {
		synchronized (controlCinema1) {
			vacanciesCinema1 += number;
			return true;
		}
	}

	public boolean returnTickets2(int number) {
		synchronized (controlCinema2) {
			vacanciesCinema2 += number;
			return true;
		}
	}

	public long getVacanciesCinema1() {
		return vacanciesCinema1;
	}

	public long getVacanciesCinema2() {
		return vacanciesCinema2;
	}
}
```

```java
public class TicketOffice1 implements Runnable {
	private Cinema cinema;

	public TicketOffice1(Cinema cinema) {
		this.cinema = cinema;
	}

	@Override
	public void run() {
		cinema.sellTickets1(3);
		cinema.sellTickets1(2);
		cinema.sellTickets2(2);
		cinema.returnTickets1(3);
		cinema.sellTickets1(5);
		cinema.sellTickets2(2);
		cinema.sellTickets2(2);
		cinema.sellTickets2(2);
	}
}
```

```java
public class TicketOffice2 implements Runnable {
	private Cinema cinema;

	public TicketOffice2(Cinema cinema) {
		this.cinema = cinema;
	}

	@Override
	public void run() {
		cinema.sellTickets2(2);
		cinema.sellTickets2(4);
		cinema.sellTickets1(2);
		cinema.sellTickets1(1);
		cinema.returnTickets2(2);
		cinema.sellTickets1(3);
		cinema.sellTickets2(2);
		cinema.sellTickets1(2);
	}
}
```

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		Cinema cinema = new Cinema();
		TicketOffice1 ticketOffice1 = new TicketOffice1(cinema);
		Thread thread1 = new Thread(ticketOffice1, "TicketOffice1");
		TicketOffice2 ticketOffice2 = new TicketOffice2(cinema);
		Thread thread2 = new Thread(ticketOffice2, "TicketOffice2");
		thread1.start();
		thread2.start();
		thread1.join();
		thread2.join();
		System.out.println("room 1 vacancies:" + cinema.getVacanciesCinema1());
		System.out.println("room 2 vacancies:" + cinema.getVacanciesCinema2());
	}
}
```

尽管只有一个影院，但是数据被拆成了两份，所也被拆成了两份，两个锁可以同时访问。并发性得到了提升。

### wait和notify的使用
条件谓语是先验条件的第一站，它在一个操作与状态之间建立起依赖关系。

每次调用wait都会隐式地与特定的条件谓词相关联。当调用特定条件谓词的wait时，调用者必须已经持有了与条件队列相关的锁，这个锁必须同时还保护着组成条件谓词的状态变量。

调用wait时必须已经持有锁，wait会释放那个锁，并阻塞当前线程，然后等待，直到特定时间的超时过后，线程被中断或者被通知唤醒。线程被唤醒后，wait会在返回运行前重新请求锁。

当使用条件等待时（Object.wait或者Condition.await）：

- 永远设置一个条件谓词——一些对象状态的测试，线程执行前必须满足它；  
- 永远在调用wait前测试条件谓词，并且从wait中返回后再次测试；  
- 永远在循环中调用wait；  
- 确保构成条件谓词的状态变量被锁保护，而这个锁正是与条件队列相关联的；  
- 当调用wait、notify或者notifyAll时，要持有与条件队列相关联的锁；并且，  
- 在检查条件谓词之后、开始执行被保护的逻辑之前，不要释放锁。  

无论何时，当你在等待一个条件，一定要确保有人会在条件谓语变为真时通知你。

在条件队列API中有两个通知方法——notify和notifyAll。无论调用哪一个，你都必须持有与条件队列对象相关联的锁。调用notify的结果是：JVM会从在这个条件队列中等待的众多线程中挑选一个，并把它唤醒：而调用notifyAll会唤醒所有正在这个条件队列中等待的线程。

大多数情况下，notifyAll都是优先于notify的选择。只有同时满足下述条件后，才能用单一的notify取代notifyAll：相同的等待者，只有一个条件谓词与条件队列相关，每个线程从wait返回后执行相同的逻辑；并且，一进一出。一个对条件变量的通知，至多激活一个线程执行。

我们实现一个生产者和消费者的例子。假设有一个生产者线程，一个消费者线程，二者同时生产和消费存储到列表中。当列表满了，阻塞生产者，等待消费者消费；当列表空了，阻塞消费者，等待生产者生产。

```java
public class Producer implements Runnable {
	private EventStorage storage;

	public Producer(EventStorage storage) {
		this.storage = storage;
	}

	@Override
	public void run() {
		for (int i = 0; i < 100; i++) {
			storage.set();
		}
	}
}
```

```java
public class Consumer implements Runnable {
	private EventStorage storage;

	public Consumer(EventStorage storage) {
		this.storage = storage;
	}

	@Override
	public void run() {
		for (int i = 0; i < 100; i++) {
			storage.get();
		}
	}
}
```

```java
import java.util.Date;
import java.util.LinkedList;
import java.util.List;

public class EventStorage {
	private int maxSize;
	private List<Date> storage;

	public EventStorage() {
		maxSize = 10;
		storage = new LinkedList<Date>();
	}

	public synchronized void set() {
		while (storage.size() == maxSize) {
			try {
				wait();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		storage.add(new Date());
		System.out.println("set: " + storage.size());
		notifyAll();
	}

	public synchronized void get() {
		while (storage.size() == 0) {
			try {
				wait();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		System.out.println("get :" + storage.size() + ":" + ((LinkedList<Date>) storage).poll());
		notifyAll();
	}
}
```

```java
public class Main {
	public static void main(String[] args) {
		EventStorage storage = new EventStorage();
		Producer producer = new Producer(storage);
		Thread thread1 = new Thread(producer);
		Consumer consumer = new Consumer(storage);
		Thread thread2 = new Thread(consumer);
		thread1.start();
		thread2.start();
	}
}
```

Object.wait会自动释放锁，并请求OS挂起当前线程，让其他线程获得该锁进而修改对象的状态。当它被唤醒时，它会在返回前重新获得锁。直观上看，调用wait意味着“我要去休息了，但是发生了需要关注的事情后叫醒我”，调用通知（notification）方法意味着“需要关注的事情发生了”。

### ReetrantLock的使用
ReetrantLock实现了Lock接口，提供了与synchronized相同的互斥和内存可见性的保证。获得ReetrantLock的锁与进入synchronized块有这相同的内存语义，释放ReetrantLock锁与退出synchronized块有相同的内存语义。

内部锁的缺陷：不能中断那些正在等待获取锁的线程，并且在请求锁失败的情况下，必须无限等待。ReetrantLock提供了更加灵活的加锁机制。

使用显式锁必须使用finally释放，否则，一旦持有锁的过程中发生异常，锁永远不会被释放。

如果你不能获得所有需要的锁，那么使用可定时的与可轮询的获取方式（tryLock）使你能够重新拿到控制权，它会释放你已经获得的这些锁，然后再重新尝试（或者至少会记录这个失败，抑或采取其他措施）。是避免死锁的有效手段。

使用内部锁一旦开始请求，锁就不能停止了。所以那些具有时间限制的活动，定时锁更适合。

当你正在响应中断的时候，lockInterruptibly方法能够使你获得锁，并且由于它是内置于Lock的，因此你不必再创建其他种类不可中断的阻塞机制。

创建可中断的锁需要两个 try 块：

```java
try{
	lock.lockInterruptibly();
	try{
		return cancelTask();
	} finally {
		lock.unlock();
	}
} finally {
	// 处理中断异常
}
```

我们模拟一个打印机，假设有多个打印任务同时使用一台打印机。

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class PrintQueue {
	private final Lock queueLock = new ReentrantLock();

	public void printJob(Object document) {
		queueLock.lock();
		Long duration = (long) (Math.random() * 10000);
		System.out.println(Thread.currentThread().getName() + ":printqueue:printing a job during " + (duration / 1000) + " seconds");
		try {
			Thread.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			queueLock.unlock();
		}
	}
}
```

```java
public class Job implements Runnable {
	private PrintQueue printQueue;

	public Job(PrintQueue printQueue) {
		this.printQueue = printQueue;
	}

	@Override
	public void run() {
		System.out.println(Thread.currentThread().getName() + ":going to print a document");
		printQueue.printJob(new Object());
		System.out.println(Thread.currentThread().getName() + ":the document has been printed");
	}
}
```

```java
public class Main {
	public static void main(String[] args) {
		PrintQueue printQueue = new PrintQueue();
		Thread[] thread = new Thread[10];
		for (int i = 0; i < 10; i++) {
			thread[i] = new Thread(new Job(printQueue), "Thread " + i);
		}
		for (int i = 0; i < 10; i++) {
			thread[i].start();
		}
	}
}
```

这里的lock具有与synchronized相同的语义。

在内部锁不能够满足使用时，ReetrantLock才被作为更高级的工具。当你需要以下高级特性时，才应该使用：可定时的、可轮询的与可中断的锁获取操作，公平队列，或者非块结构的锁。否则，请使用synchronized。

### ReentrantReadWriteLock的使用
在很多情况下，数据结构是“频繁被读取”的——它们是可变的，有的时候会被改变，但多数访问只进行读操作。如果能够放宽条件，允许多个读线程同时访问数据结构就能大大提升线程性能。读写锁就适合用于这种场景。

ReadWriteLock能够分离读写，在频繁的访问主要为读取数据结构的时候，读与写能够改进性能，在其他情况下运行的情况比独占的锁要稍差一些，这归因于它更大的复杂性。

ReentrantReadWriteLock的写入锁有一个唯一的所有者，并只能被获得了该锁的线程释放。读取锁可以维护多个读者数量。

假设我们可以设置价格也可以修改价格，多个线程可以同时读取价格，但是只能有一个线程能够修改价格。

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class PricesInfo {
	private double price1;
	private double price2;
	private ReadWriteLock lock;

	public PricesInfo() {
		price1 = 1.0;
		price2 = 2.0;
		lock = new ReentrantReadWriteLock();
	}

	public double getPrice1() {
		lock.readLock().lock();
		double value = price1;
		lock.readLock().unlock();
		return value;
	}

	public double getPrice2() {
		lock.readLock().lock();
		double value = price2;
		lock.readLock().unlock();
		return value;
	}

	public void setPrices(double price1, double price2) {
		lock.writeLock().lock();
		this.price1 = price1;
		this.price2 = price2;
		lock.writeLock().unlock();
	}
}
```

```java
public class Reader implements Runnable {
	private PricesInfo pricesInfo;

	public Reader(PricesInfo pricesInfo) {
		this.pricesInfo = pricesInfo;
	}

	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			System.out.println(Thread.currentThread().getName() + ":price 1:" + pricesInfo.getPrice1());
			System.out.println(Thread.currentThread().getName() + ":price 2:" + pricesInfo.getPrice2());
		}
	}
}
```

```java
public class Writer implements Runnable {
	private PricesInfo pricesInfo;

	public Writer(PricesInfo pricesInfo) {
		this.pricesInfo = pricesInfo;
	}

	@Override
	public void run() {
		for (int i = 0; i < 3; i++) {
			System.out.println("Writer:Attempt to modify the prices.");
			pricesInfo.setPrices(Math.random() * 10, Math.random() * 8);
			System.out.println("Writer:Prices have been modified.");
			try {
				Thread.sleep(2);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

```java
public class Main {
	public static void main(String[] args) {
		PricesInfo pricesInfo = new PricesInfo();
		Reader[] readers = new Reader[5];
		Thread[] threadsReader = new Thread[5];
		for (int i = 0; i < 5; i++) {
			readers[i] = new Reader(pricesInfo);
			threadsReader[i] = new Thread(readers[i]);
		}
		Writer writer = new Writer(pricesInfo);
		Thread threadWriter = new Thread(writer);
		for (int i = 0; i < 5; i++) {
			threadsReader[i].start();
		}
		threadWriter.start();
	}
}
```

### 锁的公平性
ReentrantLock和ReentrantReadWriteLock构造函数提供了两种公平性的选择：创建非公平锁（默认）或者公平锁。线程按顺序请求获得公平锁，然而一个非公平锁允许“闯入”：当请求这样的锁时，如果锁的状态变为可用，线程的请求可以在等待线程的队列中向前跳跃，获得该锁。在公平锁中，如果锁已经被其他线程占有，新的请求线程会加入到等待队列。

ReetrantReadWriteLock为两个锁提供了可重入的加锁语义。在公平锁中，选择权交给等待时间最长的线程：如果锁由读者获得，而一个线程请求写入锁，那么不再允许读者获得读取所，直到写者被受理，并且已经释放了写入锁。在非公平的锁中，线程允许访问的顺序是不定的。由写者降级为读者是允许的，从读者升级为写者是不允许的（会导致死锁）。

非公平锁具有比公平锁更好的性能。

前面的打印机的例子我们没有指定锁的公平性，默认是非公平的锁。为了保证先来先服务，我们可以使用公平锁取代非公平锁。

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class PrintQueue {
	private Lock queueLock = new ReentrantLock(true);

	public void printJob(Object document) {
		for (int i = 0; i < 2; i++) {
			queueLock.lock();
			Long duration = (long) (Math.random() * 10000);
			System.out.println(Thread.currentThread().getName() + ":PrintQueue: Printing a job during " + duration / 1000 + " seconds");
			try {
				Thread.sleep(duration);
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				queueLock.unlock();
			}
		}
	}
}
```

其他的代码不变，我们就将打印队列变成先来先服务的了。

### Condition的使用
前面说的synchronized和Lock都是内部锁，还有一种广义的内部条件队列，Condition。

内部条件队列有一些缺陷。每个内部锁只能有一个与之相关联的条件队列。

一个Condition和一个单独的Lock相关联，就像条件队列和单独的内部锁相关联一样：调用与Condition相关联的Lock的Lock的Lock.newCondition方法，可以创建一个Condition。Condition提供了比内部条件队列丰富得多的特征集：每个锁可以有多个等待集、可中断/不可中断的条件等待、基于时限的等待以及公平/非公平队列之间的选择。

我们可以使用锁和条件来解决生产者消费者问题。假设生产者从一个文件取出一行数据放入缓存，消费者从缓存取数据处理。

```java
public class FileMock {
	private String[] content;
	private int index;

	public FileMock(int size, int length) {
		content = new String[size];
		for (int i = 0; i < size; i++) {
			StringBuilder buffer = new StringBuilder(length);
			for (int j = 0; j < length; j++) {
				int indice = (int) Math.random() * 256;
				buffer.append((char) indice);
			}
			content[i] = buffer.toString();
		}
		index = 0;
	}

	public boolean hasMoreLines() {
		return index < content.length;
	}

	public String getLine() {
		if (this.hasMoreLines()) {
			System.out.println("Mock:" + (content.length - index));
		}
		return null;
	}
}
```

```java
import java.util.LinkedList;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class Buffer {
	private LinkedList<String> buffer;
	private int maxSize;
	private ReentrantLock lock;
	private Condition lines;
	private Condition space;
	private boolean pendingLines;

	public Buffer(int maxSize) {
		this.maxSize = maxSize;
		buffer = new LinkedList<String>();
		lock = new ReentrantLock();
		lines = lock.newCondition();
		space = lock.newCondition();
		pendingLines = true;
	}

	public void insert(String line) {
		lock.lock();
		try {
			while (buffer.size() == maxSize) {
				space.await();
			}
			buffer.offer(line);
			System.out.println(Thread.currentThread().getName() + ":inserted line: " + buffer.size());
			lines.signalAll();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public String get() {
		String line = null;
		lock.lock();
		try {
			while (buffer.size() == 0 && hasPendingLines()) {
				lines.await();
			}
			if (hasPendingLines()) {
				line = buffer.poll();
				System.out.println(Thread.currentThread().getName() + ":Line Reded:" + buffer.size());
				space.signalAll();
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
		return line;
	}

	public boolean hasPendingLines() {
		return pendingLines || buffer.size() > 0;
	}

	public void setPendingLines(boolean pendingLines) {
		this.pendingLines = pendingLines;
	}
}
```

```java
public class Producer implements Runnable {
	private FileMock mock;
	private Buffer buffer;

	public Producer(FileMock mock, Buffer buffer) {
		this.mock = mock;
		this.buffer = buffer;
	}

	@Override
	public void run() {
		buffer.setPendingLines(true);
		while (mock.hasMoreLines()) {
			String line = mock.getLine();
			buffer.insert(line);
		}
	}
}
```

```java
import java.util.Random;

public class Consumer implements Runnable {
	private Buffer buffer;

	public Consumer(Buffer buffer) {
		this.buffer = buffer;
	}

	@Override
	public void run() {
		while (buffer.hasPendingLines()) {
			String line = buffer.get();
			processLine(line);
		}
	}

	private void processLine(String line) {
		Random random = new Random();
		try {
			Thread.sleep(random.nextInt(100));
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```

```java
public class Main {
	public static void main(String[] args) {
		FileMock mock = new FileMock(100, 10);
		Buffer buffer = new Buffer(20);
		Producer producer = new Producer(mock, buffer);
		Thread threadProducer = new Thread(producer);
		Consumer[] consumers = new Consumer[3];
		Thread[] threadConsumers = new Thread[3];
		for (int i = 0; i < 3; i++) {
			consumers[i] = new Consumer(buffer);
			threadConsumers[i] = new Thread(consumers[i], "consumer " + i);
		}
		threadProducer.start();
		for (int i = 0; i < 3; i++) {
			threadConsumers[i].start();
		}
	}
}
```

危险警告：wait、notify和notifyAll在Condition对象中的对等体是await、signal和signalAll。但是，Condition继承于Object，这意味着它也有wait和notify方法。一定要确保使用了正确的版本——await和signal！