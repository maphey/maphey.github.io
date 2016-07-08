---
date: 2016-07-08 22:16:12+08:00
layout: post
title: 一起学java并发编程（十一）：信号量实践
thread: 13
categories: java
tags: java并发编程
---
### Semaphore的使用
Semaphore，即信号量，是一种计数器，用来保护一个或多个共享资源的访问。它是并发编程的一种基础工具，大多数编程语言都提供了这个机制。

如果线程要访问一个共享资源，它必须先获得信号量。如果信号量的内部计数器大于0，信号量将减1，然后允许访问这个共享资源。计数器大于0意味着有可以使用的资源，因此线程将被允许使用其中一个资源。否则，如果信号量的计数器等于0，信号量将会把线程置入休眠状态直至计数器大于0。

假设我们有一台打印机，一次只允许一个任务执行。我们使用信号量控制多任务对打印机资源的占用。

创建一个打印执行任务。

```java
public class Job implements Runnable {
	private PrintQueue printQueue;

	public Job(PrintQueue printQueue) {
		this.printQueue = printQueue;
	}

	@Override
	public void run() {
		System.out.println(Thread.currentThread().getName() + ":goging to print a job");
		printQueue.printJob(new Object());
		System.out.println(Thread.currentThread().getName() + ":The document has been printed");
	}
}
```

有一个用来保存打印任务的队列，保证一次只能有一个任务使用打印资源。

```java
import java.util.concurrent.Semaphore;

public class PrintQueue {
	private final Semaphore semaphore;

	public PrintQueue() {
		this.semaphore = new Semaphore(1);
	}

	public void printJob(Object document) {
		try {
			semaphore.acquire();
			long duration = (long) (Math.random() * 10);
			System.out.println(Thread.currentThread().getName() + ":print queue :printing a job during " + duration + " seconds");
			Thread.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			semaphore.release();
		}
	}
}
```

执行打印任务。

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

acquire()方法用来获取信号量，release()方法用来释放信号量。

信号量也有公平模式，在有资源的情况下，选择等待时间最长的那个线程。

### 使用Semaphore实现资源多副本的并发访问
假设我们有三台打印机可以共享打印任务，我们可以给信号量3个资源。

我们还是使用上节中的任务。

```java
public class Job implements Runnable {
	private PrintQueue printQueue;

	public Job(PrintQueue printQueue) {
		this.printQueue = printQueue;
	}

	@Override
	public void run() {
		System.out.println(Thread.currentThread().getName() + ":goging to print a job");
		printQueue.printJob(new Object());
		System.out.println(Thread.currentThread().getName() + ":The document has been printed");
	}
}
```

打印队列处理打印机资源的分配。

```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class PrintQueue {
	private final Semaphore semaphore;
	// 表示打印机的可用状态
	private boolean[] freePrinters;
	private Lock lockPrinters;

	public PrintQueue() {
		semaphore = new Semaphore(3);
		freePrinters = new boolean[3];
		for (int i = 0; i < 3; i++) {
			freePrinters[i] = true;
		}
		lockPrinters = new ReentrantLock();
	}

	public void printJob(Object document) {
		try {
			semaphore.acquire();
			int assignedPrinter = getPrinter();
			long duration = (long) (Math.random() * 10);
			System.out.println(Thread.currentThread().getName() + ":print queue :printing a job in printer " + assignedPrinter + " during " + duration
					+ " seconds");
			TimeUnit.SECONDS.sleep(duration);
			freePrinters[assignedPrinter] = true;
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			semaphore.release();
		}
	}

	private int getPrinter() {
		int ret = -1;
		try {
			lockPrinters.lock();
			for (int i = 0; i < freePrinters.length; i++) {
				if (freePrinters[i]) {
					ret = i;
					freePrinters[i] = false;
					break;
				}
			}
		} finally {
			lockPrinters.unlock();
		}
		return ret;
	}
}
```

执行代码同上节，这里不再赘述。

上例中，我们将信号量的计数器初始化为3，可以同时有三个线程获得对临界区的访问。

有时可能获取信号量资源时会发生阻塞，使用tryAcquire()方法可以解决这种问题。更多的关于信号量的API参见JDK doc。

### CountDownLatch的使用
CountDownLatch是Java语言提供的同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许线程一直等待。这个类使用一个整数进行初始化，这个整数就是线程要等待完成的操作的数目。当一个线程要等待某些操作先执行完时，需要调用await()方法，这个方法让线程进入休眠直到等待的所有操作都完成。当某一个操作完成后，它将调用countDown()方法将CountDownLatch类的内部计数器减1。当计数器变成0的时候，CountDownLatch类将唤醒所有调用await()方法而进去休眠的线程。

假设有一个视频会议，需要参会者都到齐之后才能开始，我们可以使用CountDownLatch实现这个需求。

每一个参会者都有自己的任务，但是任务必须在会议开始后才能进行。

```java
import java.util.concurrent.TimeUnit;

public class Participant implements Runnable {
	private Videoconference conference;
	private String name;

	public Participant(Videoconference conference, String name) {
		this.conference = conference;
		this.name = name;
	}

	@Override
	public void run() {
		long duration = (long) (Math.random() * 10);
		try {
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		conference.arrive(name);
	}
}
```

有一个视频会议，会议参与者需要指定。

```java
import java.util.concurrent.CountDownLatch;

public class Videoconference implements Runnable {
	private final CountDownLatch controller;

	public Videoconference(int number) {
		controller = new CountDownLatch(number);
	}

	public void arrive(String name) {
		System.out.println(name + " has arrived");
		controller.countDown();
		System.out.println("videoConference : waiting for " + controller.getCount() + " participants.");
	}

	@Override
	public void run() {
		System.out.println("videoConference: Initialization: " + controller.getCount() + " participants.");
		try {
			controller.await();
			System.out.println("VideoConference:All the participants have come");
			System.out.println("VideoConference: let's start...");
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

	}
}
```

执行任务。

```java
public class Main {
	public static void main(String[] args) {
		Videoconference conference = new Videoconference(10);
		Thread threadConference = new Thread(conference);
		threadConference.start();
		for (int i = 0; i < 10; i++) {
			Participant p = new Participant(conference, "Participant " + i);
			Thread t = new Thread(p);
			t.start();
		}
	}
}
```

会议调用CountDownLatch.await()进入休眠，等待CountDownLatch计数器变成0。每个参会者到达后，都会调用CountDownLatch.countDown()将计数器减1。当所有与会者都到达后，计数器为0，会议开始。

### CyclicBarrier的使用
CyclicBarrier是Java语言提供的同步辅助类，它允许多个线程在某个集合点处进行相互等待。

CyclicBarrier与CountDownLatch类似，但是也有不同之处，CyclicBarrier更加强大。CyclicBarrier类使用一个整型数进行初始化，这个数是需要在某个点上同步的线程数。当一个线程到达指定点后，它将调用await()方法等待其他线程。当线程调用await()方法后，CyclicBarrier类将阻塞这个线程并使之休眠直到所有其他线程到达。当最后一个线程调用CyclicBarrier类的await()方法时，CyclicBarrier对象将唤醒所有在等待的线程，然后这些线程将继续执行。

CyclicBarrier类有一个很有意义的改进，即它可以传入另一个Runnable对象作为初始化参数。当所有的线程都到达集合点后，CyclicBarrier类将这个Runnable对象作为线程执行。这个特性是的这个类在并行任务上可以媲美分治编程技术。

假设我们要在一个很大矩阵中搜索指定的数字，我们使用分治的方法，将搜索交给多个线程完成。当这些方法完成搜索以后，将统计所有的值出现的总次数。

我们创建一个矩阵的模拟类。

```java
import java.util.Random;

public class MatrixMock {
	private int[][] data;

	public MatrixMock(int size, int length, int number) {
		int counter = 0;
		data = new int[size][length];
		Random random = new Random();
		for (int i = 0; i < size; i++) {
			for (int j = 0; j < length; j++) {
				data[i][j] = random.nextInt(10);
				if (data[i][j] == number) {
					counter++;
				}
			}
		}
		System.out.println("Mock: There are " + counter + " ocurrences of number in generated data");
	}

	public int[] getRow(int row) {
		if (row >= 0 && row < data.length) {
			return data[row];
		}
		return null;
	}
}
```

创建一个保存搜索结果的类。

```java
public class Results {
	private int[] data;

	public Results(int size) {
		data = new int[size];
	}

	public int[] getData() {
		return data;
	}

	public void setData(int position, int value) {
		this.data[position] = value;
	}
}
```

创建一个搜索结束后需要执行的任务。

```java
public class Grouper implements Runnable {
	private Results results;

	public Grouper(Results results) {
		this.results = results;
	}

	@Override
	public void run() {
		int finalResult = 0;
		System.out.println("Grouper: processing results...");
		int[] data = results.getData();
		for (int number : data) {
			finalResult += number;
		}
		System.out.println("Grouper: total result: " + finalResult);
	}
}
```

创建搜索任务。

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class Searcher implements Runnable {
	private int firstRow;
	private int lastRow;
	private MatrixMock mock;
	private Results results;
	private int number;
	private final CyclicBarrier barrier;

	public Searcher(int firstRow, int lastRow, MatrixMock mock, Results results, int number, CyclicBarrier barrier) {
		this.firstRow = firstRow;
		this.lastRow = lastRow;
		this.mock = mock;
		this.results = results;
		this.number = number;
		this.barrier = barrier;
	}

	@Override
	public void run() {
		int counter;
		System.out.println(Thread.currentThread().getName() + ": processing lines from " + firstRow + " to " + lastRow);
		for (int i = firstRow; i < lastRow; i++) {
			int[] row = mock.getRow(i);
			counter = 0;
			for (int j = 0; j < row.length; j++) {
				if (row[j] == number) {
					counter++;
				}
			}
			results.setData(i, counter);
		}
		System.out.println(Thread.currentThread().getName() + ": Lines processed.");
		try {
			barrier.await();
		} catch (InterruptedException | BrokenBarrierException e) {
			e.printStackTrace();
		}
	}
}
```

执行，并查看结果。

```java
import java.util.concurrent.CyclicBarrier;

public class Main {
	public static void main(String[] args) {
		final int ROWS = 10000;
		final int NUMBERS = 1000;
		final int SEARCH = 5;
		final int PARTICIPANTS = 5;
		final int LINES_PARTICIPANT = 2000;

		MatrixMock mock = new MatrixMock(ROWS, NUMBERS, SEARCH);
		Results results = new Results(ROWS);
		Grouper grouper = new Grouper(results);
		CyclicBarrier barrier = new CyclicBarrier(PARTICIPANTS, grouper);
		Searcher[] searchers = new Searcher[PARTICIPANTS];
		for (int i = 0; i < PARTICIPANTS; i++) {
			searchers[i] = new Searcher(i * LINES_PARTICIPANT, (i * LINES_PARTICIPANT) + LINES_PARTICIPANT, mock, results, 5, barrier);
			Thread thread = new Thread(searchers[i]);
			thread.start();
		}
		System.out.println("Main: the main thread has finished.");
	}
}
```

CyclicBarrier提供了reset()方法，可以重置对象。当重置发生后，在await()方法中等待的线程将收到一个BrokenBarrierException异常。

CyclicBarrier对象有一种特殊的状态即损坏状态。当很多线程在await()方法上等待的时候，如果其中一个线程被中断，这个线程将抛出InterruptedException异常，其他等待的线程将抛出BrokenBarrierException异常。

### Phaser的使用
Phaser是Java语言提供的同步辅助类。它把并发任务分成多个阶段运行，在开始下一阶段之前，当前阶段中的所有线程都必须执行完成。

Phaser可以动态的增加或者减少任务数。

假设我们我们有一个分为3步的任务：

1. 在指定的文件夹及子文件夹查找.log文件，如果没有结果，任务就结束；  
2. 对第一步的结果进行过滤，删除修改时间超过24小时的文件，如果没有结果，任务就结束；  
3. 将结果打印到控制台。  

```java
import java.io.File;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.concurrent.Phaser;
import java.util.concurrent.TimeUnit;

public class FileSearch implements Runnable {
	private String initPath;
	private String end;
	private List<String> results;
	private Phaser phaser;

	public FileSearch(String initPath, String end, Phaser phaser) {
		this.initPath = initPath;
		this.end = end;
		this.phaser = phaser;
		this.results = new ArrayList<String>();
	}

	@Override
	public void run() {
		phaser.arriveAndAwaitAdvance();
		System.out.println(Thread.currentThread().getName() + " :starting.");
		File file = new File(initPath);
		if (file.isDirectory()) {
			directoryProcess(file);
		}
		if (!checkResults()) {
			return;
		}
		filterResults();
		if (!checkResults()) {
			return;
		}
		showInfo();
		phaser.arriveAndDeregister();
		System.out.println(Thread.currentThread().getName() + " work completed.");
	}

	private void directoryProcess(File file) {
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
	}

	private void fileProcess(File file) {
		if (file.getName().endsWith(end)) {
			results.add(file.getAbsolutePath());
		}
	}

	private void filterResults() {
		List<String> newResults = new ArrayList<String>();
		long actualDate = new Date().getTime();
		for (int i = 0; i < results.size(); i++) {
			File file = new File(results.get(i));
			long fileDate = file.lastModified();
			if (actualDate - fileDate < TimeUnit.MILLISECONDS.convert(1, TimeUnit.DAYS)) {
				newResults.add(results.get(i));
			}
		}
		results = newResults;
	}

	private boolean checkResults() {
		if (results.isEmpty()) {
			System.out.println(Thread.currentThread().getName() + ": phase " + phaser.getPhase() + " :0 results.");
			System.out.println(Thread.currentThread().getName() + ": phase " + phaser.getPhase() + " :End.");
			phaser.arriveAndDeregister();
			return false;
		} else {
			System.out.println(Thread.currentThread().getName() + ": phase " + phaser.getPhase() + ": " + results.size() + " results.");
			phaser.arriveAndAwaitAdvance();
			return true;
		}
	}

	private void showInfo() {
		for (int i = 0; i < results.size(); i++) {
			File file = new File(results.get(i));
			System.out.println(Thread.currentThread().getName() + " : " + file.getAbsolutePath());
		}
		phaser.arriveAndAwaitAdvance();
	}
}
```

执行任务并查看结果。

```java
import java.util.concurrent.Phaser;

public class Main {
	public static void main(String[] args) {
		Phaser phaser = new Phaser(3);
		FileSearch system = new FileSearch("C:\\Windows", "log", phaser);
		FileSearch apps = new FileSearch("C:\\Program Files", "log", phaser);
		FileSearch documents = new FileSearch("C:\\Documents And Settings", "log", phaser);
		Thread systemThread = new Thread(system, "System");
		systemThread.start();
		Thread appsThread = new Thread(apps, "Apps");
		appsThread.start();
		Thread documentsThread = new Thread(documents, "Documents");
		documentsThread.start();
		try {
			systemThread.join();
			appsThread.join();
			documentsThread.join();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("Terminated: " + phaser.isTerminated());
	}
}
```

Phaser初始化有个线程数，当一个Phaser对象调用arriveAndAwaitAdvance()方法，Phaser对象将减1，并把这个线程置于休眠状态，直到所有的线程完成这个阶段。在run()方法的开头调用这个方法可以保障在所有线程创建好之前没有线程开始执行任务。

在第二阶段，如果结果集是空的，对应的线程没有理由继续执行，所以返回；但是必须通知Phaser对象参与同步的线程少了一个。通过调用arriveAndDeregister()方法告诉Phaser对象，这个线程已经完成，不会参与下一阶段。

在第三阶段结束后，在showInfo()中调用了Phaser对象的arriveAndAwaitAdvance()方法。通过这个调用确保三个线程都已完成。showInfo()方法完成后，调用arriveAndDeregister()方法，撤销Phaser中线程的注册。所以当所有线程结束后，Phaser对象就没有参与同步的线程了。

Phaser提供了两种方法增加注册者数量：register()新增一个参与者，bulkRegister(int parties)新增指定数目的参与者。

### Phaser实现并发阶段任务的阶段切换
Phaser类提供了onAdvance()方法，它在Phaser阶段改变的时候会被自动执行。onAdvance()方法需要两个int型的传入参数：当前阶段数以及注册的参与者数量。它返回的是boolean值，如果返回false表示Phaser在继续执行，返回true表示Phaser已经完成执行并且进入了终止态。

这个方法默认实现如下：如果注册的参与者数量是0就返回true，否则反回false。但是我们通过继承Phaser类覆盖这个方法。一般来说，当必须在一个阶段到另一个阶段过度的时候执行一些操作，那么我们就得这么做。

假设学生需要完成3道试题，只有所有学生都完成一道试题后才开始下一题，每次开始下一题都进行一些处理。

我们覆盖Phaser的onAdvance()方法，每阶段结束后进行一些处理。

```java
import java.util.concurrent.Phaser;

public class MyPhaser extends Phaser {

	@Override
	protected boolean onAdvance(int phase, int registeredParties) {
		switch (phase) {
		case 0:
			return studentsArrived();
		case 1:
			return finishFirstExercise();
		case 2:
			return finishSecondExercise();
		case 3:
			return finishExam();
		default:
			return true;
		}
	}

	private boolean studentsArrived() {
		System.out.println("phaser: the exam are going to start. the students are ready.");
		System.out.println("phaser: we have " + getRegisteredParties() + " students.");
		return false;
	}

	private boolean finishFirstExercise() {
		System.out.println("phaser: all the students have finished the first exercise.");
		System.out.println("pahser: it's time for the second one.");
		return false;
	}

	private boolean finishSecondExercise() {
		System.out.println("phaser: all the students have finished the second exercise.");
		System.out.println("phaser: it's time for the third one.");
		return false;
	}

	private boolean finishExam() {
		System.out.println("phaser: all the students have finished the exam.");
		System.out.println("phaser: thank you for your time.");
		return true;
	}
}
```

学生线程，参与3道试题。

```java
import java.util.Date;
import java.util.concurrent.Phaser;
import java.util.concurrent.TimeUnit;

public class Student implements Runnable {
	private Phaser phaser;

	public Student(Phaser phaser) {
		this.phaser = phaser;
	}

	@Override
	public void run() {
		System.out.println(Thread.currentThread().getName() + ": has arrived to do the exam." + new Date());
		phaser.arriveAndAwaitAdvance();
		System.out.println(Thread.currentThread().getName() + ": is goging to do the first exercise." + new Date());
		doExercise1();
		System.out.println(Thread.currentThread().getName() + ": has done the first exercise." + new Date());
		phaser.arriveAndAwaitAdvance();
		System.out.println(Thread.currentThread().getName() + ": is goging to do the second exercise." + new Date());
		doExercise2();
		phaser.arriveAndAwaitAdvance();
		System.out.println(Thread.currentThread().getName() + ": is goging to do the third exercise." + new Date());
		doExercise3();
		System.out.println(Thread.currentThread().getName() + ": has finished the exam." + new Date());
		phaser.arriveAndAwaitAdvance();
	}

	private void doExercise1() {
		long duration = (long) Math.random() * 10;
		try {
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	private void doExercise2() {
		long duration = (long) Math.random() * 20;
		try {
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	private void doExercise3() {
		long duration = (long) Math.random() * 30;
		try {
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```

执行并查看结果。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		MyPhaser phaser = new MyPhaser();
		Student[] students = new Student[5];
		for (int i = 0; i < students.length; i++) {
			students[i] = new Student(phaser);
			phaser.register();
		}
		Thread[] threads = new Thread[students.length];
		for (int i = 0; i < threads.length; i++) {
			threads[i] = new Thread(students[i], "student " + i);
			threads[i].start();
		}
		for (int i = 0; i < threads.length; i++) {
			threads[i].join();
		}
		System.out.println("main: the phaser has finished: " + phaser.isTerminated());
	}
}
```

这里MyPhaser没有指定参与者的数目，但是每个学生对象调用了Phaser的register()方法，在Phaser中注册。

### Exchanger的使用
Exchanger是Java语言提供的同步辅助类。它提供了两个线程之间的数据交换点。当两个线程都到达同步点时，它们交换数据结构，因此第一个线程的数据结构进入到第二个线程中，同时第二个线程的数据结构进入到第一个线程中。

Exchanger只能用于两个线程，可以用来解决一对一的生产者-消费者问题。

假设有一个生产者。

```java
import java.util.List;
import java.util.concurrent.Exchanger;

public class Producer implements Runnable {
	private List<String> buffer;
	private final Exchanger<List<String>> exchanger;

	public Producer(List<String> buffer, Exchanger<List<String>> exchanger) {
		this.buffer = buffer;
		this.exchanger = exchanger;
	}

	@Override
	public void run() {
		int cycle = 1;
		for (int i = 0; i < 10; i++) {
			System.out.println("producer: cycle " + cycle);
			for (int j = 0; j < 10; j++) {
				String message = "Event: " + ((i * 10) + j);
				System.out.println("Producer: " + message);
				buffer.add(message);
			}
			try {
				buffer = exchanger.exchange(buffer);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println("Producer: " + buffer.size());
			cycle++;
		}
	}
}
```

有一个消费者。

```java
import java.util.List;
import java.util.concurrent.Exchanger;

public class Consumer implements Runnable {
	private List<String> buffer;
	private final Exchanger<List<String>> exchanger;

	public Consumer(List<String> buffer, Exchanger<List<String>> exchanger) {
		this.buffer = buffer;
		this.exchanger = exchanger;
	}

	@Override
	public void run() {
		int cycle = 1;
		for (int i = 0; i < 10; i++) {
			System.out.println("consumer: cycle " + cycle);
			try {
				buffer = exchanger.exchange(buffer);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println("consumer: " + buffer.size());
			for (int j = 0; j < 10; j++) {
				String message = buffer.get(0);
				System.out.println("consumer:" + message);
				buffer.remove(0);
			}
			cycle++;
		}
	}
}
```

运行并查看结果。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Exchanger;

public class Main {
	public static void main(String[] args) {
		List<String> buffer1 = new ArrayList<String>();
		List<String> buffer2 = new ArrayList<String>();
		Exchanger<List<String>> exchanger = new Exchanger<List<String>>();
		Producer producer = new Producer(buffer1, exchanger);
		Consumer consumer = new Consumer(buffer2, exchanger);
		Thread threadProducer = new Thread(producer);
		Thread threadConsumer = new Thread(consumer);
		threadProducer.start();
		threadConsumer.start();
	}
}
```

第一个调用exchange()的线程会被置于休眠，直到其他的线程到达后，完成数据交换。