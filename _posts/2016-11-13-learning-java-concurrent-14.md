---
date: 2016-11-13 22:16:12+08:00
layout: post
title: 一起学java并发编程（十四）：并发集合的实践
thread: 16
categories: java
tags: java并发编程
---
数据结构是编程的基本元素，Java集合框架，为编程提供了许多现成的数据结构。并发编程对数据结构有特殊的要求，在并发程序中使用数据集合时，必须谨慎地选择实现方式。大部分集合不能直接用于并发应用，因为它们没有对本身数据的并发访问控制。

Java为并发程序提供了两套集合框架：

- 阻塞式集合。这类集合在添加或移除元素时，会对集合进行判断。当集合已满或者为空时，对集合进行添加或移除就会阻塞执行线程，直到添加或移除操作被成功执行。
- 非阻塞式集合。这类集合也包括添加和移除数据的方法。如果方法不能立即被执行，则返回null或抛出异常，但是调用这个方法的线程不会被阻塞。

### ConcurrentLinkedDeque
ConcurrentLinkedDeque是非阻塞式线程安全双向队列，可以从队列的两端对元素进行操作。

模拟队列添加10000个元素。

```java
public class AddTask implements Runnable {
	private ConcurrentLinkedDeque<String> list;

	public AddTask(ConcurrentLinkedDeque<String> list) {
		this.list = list;
	}

	@Override
	public void run() {
		String name = Thread.currentThread().getName();
		for (int i = 0; i < 10000; i++) {
			list.add(name + ":element " + i);
		}
	}
}
```

模拟从队列中取出10000个元素。

```java
public class PollTask implements Runnable {
	private ConcurrentLinkedDeque<String> list;

	public PollTask(ConcurrentLinkedDeque<String> list) {
		this.list = list;
	}

	@Override
	public void run() {
		for (int i = 0; i < 5000; i++) {
			list.pollFirst();
			list.pollLast();
		}
	}
}
```

定义100个线程，向对列中新增和移除元素，并查看最终结果。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		ConcurrentLinkedDeque<String> list = new ConcurrentLinkedDeque<String>();
		Thread[] threads = new Thread[100];
		for (int i = 0; i < threads.length; i++) {
			AddTask task = new AddTask(list);
			threads[i] = new Thread(task);
			threads[i].start();
		}
		System.out.println("main: " + threads.length + " addtask threads have been launched");
		for (int i = 0; i < threads.length; i++) {
			threads[i].join();
		}
		System.out.println("main: size of the list:" + list.size());
		for (int i = 0; i < threads.length; i++) {
			PollTask task = new PollTask(list);
			threads[i] = new Thread(task);
			threads[i].start();
		}
		System.out.println("main: " + threads.length + " pooltask threads have been launched");
		for (int i = 0; i < threads.length; i++) {
			threads[i].join();
		}
		System.out.println("main: size of the list:" + list.size());
	}
}
```

如果此时在队列满的时候调用add()等添加元素的方法，或者在队列空的时候调用peek()等获取元素的方法，会抛出IllegalStateException异常或者NoSuchElementException异常。

### LinkedBlockingDeque
LinkedBlockingDeque是线程安全的阻塞队列。它与ConcurrentLinkedDeque的主要区别是，阻塞队列在插入和删除操作时，如果队列已满或为空，操作不会被立即执行，而是将调用这个操作的线程阻塞，直到操作可以执行成功。

假设我们每两秒向队列添加5个元素，重复3次。

```java
public class Client implements Runnable {
	private LinkedBlockingDeque<String> requestList;

	public Client(LinkedBlockingDeque<String> requestList) {
		this.requestList = requestList;
	}

	@Override
	public void run() {
		for (int i = 0; i < 3; i++) {
			for (int j = 0; j < 5; j++) {
				StringBuffer request = new StringBuffer();
				request.append(i);
				request.append(":");
				request.append(j);
				try {
					requestList.put(request.toString());
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				System.out.println("client: " + request + " at " + new Date());
			}
			try {
				TimeUnit.SECONDS.sleep(2);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		System.out.println("client: End");
	}
}
```

每300毫秒从队列中取出3个字符串，重复5次。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		LinkedBlockingDeque<String> list = new LinkedBlockingDeque<String>(3);
		Client client = new Client(list);
		Thread thread = new Thread(client);
		thread.start();
		for (int i = 0; i < 5; i++) {
			for (int j = 0; j < 3; j++) {
				String request = list.take();
				System.out.println("main: request: " + request + " at " + new Date() + " .Size: " + list.size());
			}
			TimeUnit.MILLISECONDS.sleep(300);
		}
		System.out.println("main: end of the program.");
	}
}
```

这时Client类中的put方法在队列满的时候，会阻塞队列，直到队列有可用空间。Main类中的take()方法会在队列为空的时候等待队列不为空。

### PriorityBlockingQueue
为了解决线程优先级的问题，Java定义了PriorityBlockingQueue队列，用于存放有优先级的元素，这样就可以按照优先级处理队列中的元素。
添加进PriorityBlockingQueue的元素必须实现Comparable接口。这样队列中的元素就可以按照compareTo()方法返回结果进行排序。

而且，PriorityBlockingQueue是一个阻塞队列，当方法不能被立即执行时，就会阻塞到方法执行成功。

假设我们相队列中存放Event元素，并实现Comparable接口，用来比较priority属性。

```java
public class Event implements Comparable<Event> {
	private int thread;
	private int priority;

	public Event(int thread, int priority) {
		this.thread = thread;
		this.priority = priority;
	}

	public int getThread() {
		return thread;
	}

	public int getPriority() {
		return priority;
	}

	@Override
	public int compareTo(Event o) {
		if (this.priority > o.getPriority()) {
			return -1;
		} else if (this.priority < o.getPriority()) {
			return 1;
		}
		return 0;
	}
}
```

实现Runnable接口，实现向队列中存放1000个Event元素，并设置元素的优先级。

```java
public class Task implements Runnable {
	private int id;
	private PriorityBlockingQueue<Event> queue;

	public Task(int id, PriorityBlockingQueue<Event> queue) {
		this.id = id;
		this.queue = queue;
	}

	@Override
	public void run() {
		for (int i = 0; i < 1000; i++) {
			Event event = new Event(id, i);
			queue.add(event);
		}
	}
}
```
我们创建5个线程执行添加任务，这样一共产生了5000个元素。输出队列可以发现队列中的元素按照优先级排序了。

### DelayQueue
Java还提供了一种延迟执行的元素队列DelayQueue。存放到DelayQueue中的元素必须实现Delayed接口，Delayed接口使对象成为延迟对象，它使存放在DelayQueue中的对象有了激活日期。Delayed接口会强制实现compareTo()方法和getDelay()方法。compareTo()方法用于按照延迟时间进行排序，getDelay()方法返回到激活日期的剩余时间。

实现Delayed接口。

```java
public class Event implements Delayed {
	private Date startDate;

	public Event(Date startDate) {
		this.startDate = startDate;
	}

	@Override
	public int compareTo(Delayed o) {
		long result = this.getDelay(TimeUnit.NANOSECONDS) - o.getDelay(TimeUnit.NANOSECONDS);
		if (result < 0) {
			return -1;
		} else if (result > 0) {
			return 1;
		}
		return 0;
	}

	@Override
	public long getDelay(TimeUnit unit) {
		Date now = new Date();
		long diff = startDate.getTime() - now.getTime();
		return unit.convert(diff, TimeUnit.MILLISECONDS);
	}
}
```

实现Runnable接口，用于创建延迟队列。

```java
public class Task implements Runnable {
	private int id;
	private DelayQueue<Event> queue;

	public Task(int id, DelayQueue<Event> queue) {
		this.id = id;
		this.queue = queue;
	}

	@Override
	public void run() {
		Date now = new Date();
		Date delay = new Date();
		delay.setTime(now.getTime() + (id * 1000));
		System.out.println("thread " + id + " : " + delay);
		for (int i = 0; i < 100; i++) {
			Event event = new Event(delay);
			queue.add(event);
		}
	}
}
```

创建5个线程，向队列添加任务，并获取队列中的元素。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		DelayQueue<Event> queue = new DelayQueue<Event>();
		Thread[] threads = new Thread[5];
		for (int i = 0; i < threads.length; i++) {
			Task task = new Task(i + 1, queue);
			threads[i] = new Thread(task);
		}
		for (int i = 0; i < threads.length; i++) {
			threads[i].start();
		}
		for (int i = 0; i < threads.length; i++) {
			threads[i].join();
		}
		do {
			int counter = 0;
			Event event;
			do {
				event = queue.poll();
				if (event != null) {
					counter++;
				}
			} while (event != null);
			System.out.println("at " + new Date() + " you have read " + counter + " events");
		} while (queue.size() > 0);
	}
}
```

可以看到，当队列中有激活元素的时候，poll()方法才返回元素，否则返回null。

### ConcurrentNavigableMap
ConcurrentNavigableMap是一个有序Map接口，可以实现以递归方式支持其可导航子映射的ConcurrentMap操作。

ConcurrentSkipListMap是ConcurrentNavigableMap的一个实现类，是非阻塞的并发容器。

我们实现一个联系人列表，假设每个联系人由名字和电话组成。

```java
public class Contact {
	private String name;
	private String phone;

	public String getName() {
		return name;
	}

	public String getPhone() {
		return phone;
	}

	public Contact(String name, String phone) {
		this.name = name;
		this.phone = phone;
	}
}
```

我们以联系人的姓名和电话为key，存放联系人联系方式。

```java
public class Task implements Runnable {
	private ConcurrentSkipListMap<String, Contact> map;
	private String id;

	public Task(ConcurrentSkipListMap<String, Contact> map, String id) {
		this.map = map;
		this.id = id;
	}

	@Override
	public void run() {
		for (int i = 0; i < 1000; i++) {
			Contact contact = new Contact(id, String.valueOf(i + 1000));
			map.put(id + contact.getPhone(), contact);
		}
	}
}
```

创建25个线程，用来生成ConcurrentSkipListMap信息。我们可以获取第一个联系人信息，最后一个联系人信息，或者按照key的范围获取一个Map子集。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		ConcurrentSkipListMap<String, Contact> map = new ConcurrentSkipListMap<String, Contact>();
		Thread[] threads = new Thread[25];
		int counter = 0;
		for (char i = 'A'; i < 'Z'; i++) {
			Task task = new Task(map, String.valueOf(i));
			threads[counter] = new Thread(task);
			threads[counter].start();
			counter++;
		}
		for (int i = 0; i < 25; i++) {
			threads[i].join();
		}
		System.out.println("main: size of the map:" + map.size());
		Map.Entry<String, Contact> element = map.firstEntry();
		Contact contact = element.getValue();
		System.out.println("main: first entry: " + contact.getName() + ":" + contact.getPhone());
		element = map.lastEntry();
		contact = element.getValue();
		System.out.println("main: last entry:" + contact.getName() + ":" + contact.getPhone());
		System.out.println("main: submap from A1996 to B1002:");
		ConcurrentNavigableMap<String, Contact> submap = map.subMap("A1996", "B1002");
		do {
			element = submap.pollFirstEntry();
			if (element != null) {
				contact = element.getValue();
				System.out.println(contact.getName() + ":" + contact.getPhone());
			}
		} while (element != null);
	}
}
```

SkipList是一种高效的排序数据结构，具有高效和节省空间的双重优点。

### ThreadLocalRandom
ThreadLocalRandom类是一个本地变量。每个生成随机数的线程都有一个不同的生成器，但是都在同一个类中被管理。相对于使用共享的Random对象，这种机制在并发情况下有更好的性能。

这个类的使用方式非常简单。

```java
public class TaskLocalRandom implements Runnable {

	@Override
	public void run() {
		String name = Thread.currentThread().getName();
		for (int i = 0; i < 10; i++) {
			System.out.println(name + ": " + ThreadLocalRandom.current().nextInt());
		}
	}
}
```

观察并发环境下生成的随机数。

```java
public class Main {
	public static void main(String[] args) {
		Thread[] threads = new Thread[3];
		for (int i = 0; i < 3; i++) {
			TaskLocalRandom task = new TaskLocalRandom();
			threads[i] = new Thread(task);
			threads[i].start();
		}
	}
}
```

### 原子变量
在并发环境下，当多个线程共享同一个变量时，就会发生数据不一致。为了避免这种错误，Java5引入了原子变量。当一个线程在对原子变量操作时，如果其他线程也试图对同一原子变量执行操作，原子变量的实现类提供了一套机制来检查操作是否在一步内完成。这套机制其实就是CAS操作。

现代很多CPU都支持CAS操作指令，使用这种方式能够提高多线程并发操作变量的效率。

我们模拟一个银行帐号，使用原子变量AtomicLong定义账户余额。

```java
public class Account {
	private AtomicLong balance;

	public Account() {
		balance = new AtomicLong();
	}

	public long getBalance() {
		return balance.get();
	}

	public void setBalance(long balance) {
		this.balance.getAndSet(balance);
	}

	public void addAmount(long amount) {
		balance.getAndAdd(amount);
	}

	public void subtractAmount(long amount) {
		this.balance.getAndAdd(-amount);
	}
}
```

定义一个Company，可以对账户进行操作。

```java
public class Company implements Runnable {
	private Account account;

	public Company(Account account) {
		this.account = account;
	}

	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			account.addAmount(1000);
		}
	}
}
```

定义Bank类，也可以对账户进行操作。

```java
public class Bank implements Runnable {
	private Account account;

	public Bank(Account account) {
		this.account = account;
	}

	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			account.subtractAmount(1000);
		}
	}
}
```

可以看到，在并发情况下，原子变量可以保证多线程对原子变量的并发安全性。

```java
public class Main {
	public static void main(String[] args) {
		Account account = new Account();
		account.setBalance(1000);
		Company company = new Company(account);
		Thread companyThread = new Thread(company);
		Bank bank = new Bank(account);
		Thread bankThread = new Thread(bank);
		System.out.println("account: initial balance:" + account.getBalance());
		companyThread.start();
		bankThread.start();
		try {
			companyThread.join();
			bankThread.join();
			System.out.println("account: final balance:" + account.getBalance());
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```

### 原子数组
除了引用原子变量，Java还提供了原子数组，同样也是基于CAS操作。

我们分别定义一个增加元素的类和一个减少元素的类，对AtomicIntegerArray进行并发操作。

```java
public class Incrementer implements Runnable {
	private AtomicIntegerArray vector;

	public Incrementer(AtomicIntegerArray vector) {
		this.vector = vector;
	}

	@Override
	public void run() {
		for (int i = 0; i < vector.length(); i++) {
			vector.getAndIncrement(i);
		}
	}
}
```

```java
public class Decrementer implements Runnable {
	private AtomicIntegerArray vector;

	public Decrementer(AtomicIntegerArray vector) {
		this.vector = vector;
	}

	@Override
	public void run() {
		for (int i = 0; i < vector.length(); i++) {
			vector.getAndDecrement(i);
		}
	}
}
```

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		final int THREADS = 100;
		AtomicIntegerArray vector = new AtomicIntegerArray(1000);
		Incrementer incrementer = new Incrementer(vector);
		Decrementer decrementer = new Decrementer(vector);
		Thread[] threadIncrementer = new Thread[THREADS];
		Thread[] threadDecrementer = new Thread[THREADS];
		for (int i = 0; i < THREADS; i++) {
			threadIncrementer[i] = new Thread(incrementer);
			threadDecrementer[i] = new Thread(decrementer);
			threadIncrementer[i].start();
			threadDecrementer[i].start();
		}
		for (int i = 0; i < THREADS; i++) {
			threadIncrementer[i].join();
			threadDecrementer[i].join();
		}
		for (int i = 0; i < vector.length(); i++) {
			if (vector.get(i) != 0) {
				System.out.println("vector[" + i + "]:" + vector.get(i));
			}
		}
		System.out.println("main: end of the example");
	}
}
```

使用原子数组可以简化并发环境下对变量的控制，同时能够配合JVM提供更好的性能。