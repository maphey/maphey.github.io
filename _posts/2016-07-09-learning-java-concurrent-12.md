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
我们使用submit方法将Callable对象发送给执行器执行，会返回Future对象。Future对象可以用于以下两个主要目的。

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
有时候我们会用多个任务来解决一个问题，往往只关心这些任务中的第一个结果。

假如我们需要验证用户登录权限，我们有LDAP验证和数据库验证两种方式。如果任意一种通过验证，则表示授权成功。

创建一个用户验证类：

```java
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class UserValidator {
	private String name;

	public UserValidator(String name) {
		this.name = name;
	}

	public String getName() {
		return name;
	}

	public boolean validate(String name, String password) {
		Random random = new Random();
		long duration = (long) Math.random() * 10;
		System.out.println("validator " + this.name + ": validating a user during " + duration + " seconds");
		try {
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			return false;
		}
		return random.nextBoolean();
	}
}
```

创建验证任务，为了获取结果使用Callable接口：

```java
import java.util.concurrent.Callable;

public class TaskValidator implements Callable<String> {
	private UserValidator validator;
	private String user;
	private String password;

	public TaskValidator(UserValidator validator, String user, String password) {
		this.validator = validator;
		this.user = user;
		this.password = password;
	}

	@Override
	public String call() throws Exception {
		if (!validator.validate(user, password)) {
			System.out.println(validator.getName() + ": the user has not found");
			throw new RuntimeException("Error validating user");
		}
		System.out.println(validator.getName() + ": the user has been found.");
		return validator.getName();
	}
}
```

我们可以创建一个线程池，调用invokeAny方法，返回第一个成功执行的结果。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Main {
	public static void main(String[] args) {
		String username = "test";
		String password = "test";
		UserValidator ldapValidator = new UserValidator("LDAP");
		UserValidator dbValidator = new UserValidator("DataBase");
		TaskValidator ldapTask = new TaskValidator(ldapValidator, username, password);
		TaskValidator dbTask = new TaskValidator(dbValidator, username, password);
		List<TaskValidator> taskList = new ArrayList<TaskValidator>();
		taskList.add(ldapTask);
		taskList.add(dbTask);
		ExecutorService executor = Executors.newCachedThreadPool();
		String result;
		try {
			result = executor.invokeAny(taskList);
			System.out.println("Main: result: " + result);
		} catch (InterruptedException | ExecutionException e) {
			e.printStackTrace();
		}
		executor.shutdown();
		System.out.println("main: end of the execution");
	}
}
```

### 使用ExecutorService返回所有结果

在上面我们通过遍历的方法获取了任务的执行结果。其实ExecutorService还有一个invokeAll方法可以直接获取所有的任务执行结果。

假设我们需要的任务执行结果如下：

```java
public class Result {
	private String name;
	private int value;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getValue() {
		return value;
	}

	public void setValue(int value) {
		this.value = value;
	}
}
```

同样，我们需要实现能够返回结果的任务Callable。

```java
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

public class Task implements Callable<Result> {
	private String name;

	public Task(String name) {
		this.name = name;
	}

	@Override
	public Result call() throws Exception {
		System.out.println(name + ": starting");
		long duration = (long) (Math.random() * 10);
		System.out.println(name + ":waiting " + duration + " seconds for results");
		TimeUnit.SECONDS.sleep(duration);
		int value = 0;
		for (int i = 0; i < 5; i++) {
			value += (int) Math.random() * 100;
		}
		Result result = new Result();
		result.setName(name);
		result.setValue(value);
		System.out.println(name + ": Ends");
		return result;
	}
}
```

创建任务并执行：

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class Main {
	public static void main(String[] args) throws InterruptedException, ExecutionException {
		ExecutorService executor = Executors.newCachedThreadPool();
		List<Task> taskList = new ArrayList<Task>();
		for (int i = 0; i < 3; i++) {
			Task task = new Task(String.valueOf(i));
			taskList.add(task);
		}
		List<Future<Result>> resultList = executor.invokeAll(taskList);
		executor.shutdown();
		System.out.println("main: printing the results");
		for (int i = 0; i < resultList.size(); i++) {
			Future<Result> future = resultList.get(i);
			Result result = future.get();
			System.out.println(result.getName() + ":" + result.getValue());
		}
	}
}
```

### 使用ScheduledThreadPoolExecutor执行延时任务

有时候我们需要定时任务或者延时任务，尽管有Quartz这种专门用于定时任务处理的框架，Java也提供了内置的定时任务处理方法。

假如我们有一个任务需要执行。

```java
import java.util.Date;
import java.util.concurrent.Callable;

public class Task implements Callable<String> {
	private String name;

	public Task(String name) {
		this.name = name;
	}

	@Override
	public String call() throws Exception {
		System.out.println(name + ": starting at:" + new Date());
		return "hello, world";
	}
}
```

我们可以直接使用ScheduledThreadPoolExecutor对象实现。我们发送一批任务到执行框架，实现每隔1秒有一个任务开始执行。

```java
import java.util.Date;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) throws InterruptedException {
		ScheduledThreadPoolExecutor executor = (ScheduledThreadPoolExecutor) Executors.newScheduledThreadPool(1);
		System.out.println("main: starting at:" + new Date());
		for (int i = 0; i < 5; i++) {
			Task task = new Task("task " + i);
			executor.schedule(task, i + 1, TimeUnit.SECONDS);
		}
		executor.shutdown();
		executor.awaitTermination(1, TimeUnit.DAYS);
	}
}
```

### 使用ScheduledThreadPoolExecutor执行周期任务

上面我们展示一个延时任务，我们再展示一个定时任务。

定义一个任务类。

```java
import java.util.Date;

public class Task implements Runnable {
	private String name;

	public Task(String name) {
		this.name = name;
	}

	@Override
	public void run() {
		System.out.println(name + ": starting at:" + new Date());
	}
}
```

使用ScheduledThreadPoolExecutor执行scheduleAtFixedRate方法。

```java
import java.util.Date;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) throws Exception {
		ScheduledThreadPoolExecutor executor = (ScheduledThreadPoolExecutor) Executors.newScheduledThreadPool(1);
		System.out.println("main: starting at:" + new Date());
		Task task = new Task("task");
		ScheduledFuture<?> result = executor.scheduleAtFixedRate(task, 1, 2, TimeUnit.SECONDS);
		for (int i = 0; i < 10; i++) {
			System.out.println("main: delay:" + result.getDelay(TimeUnit.MILLISECONDS));
			TimeUnit.MILLISECONDS.sleep(500);
		}
		executor.shutdown();
		TimeUnit.SECONDS.sleep(5);
		System.out.println("main: finished at:" + new Date());
	}
}
```

scheduleAtFixedRate方法会根据延迟的时间、固定执行的频率执行任务。但是一定要注意，如果任务执行的间隔期短于任务执行是时间，最终会堆积很多任务，导致服务器负载过高。

### 取消执行器中的任务
Executors的子接口ExecutorService暗示了生命周期有3中状态：运行、关闭和终止。ExecutorService最初创建后的初始状态是运行状态。shutdown方法会启动一个平缓的关闭过程：停止接收新的任务，同时等待已经提交的任务完成——包括尚未开始执行的任务。shutdownNow方法会启动一个强制关闭过程：尝试取消所有运行中的任务和排在队列中尚未开始的任务。

```java
import java.util.concurrent.Callable;

public class Task implements Callable<String> {

	@Override
	public String call() throws Exception {
		while (true) {
			System.out.println("task:test");
			Thread.sleep(100);
		}
	}
}
```

```java
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) throws InterruptedException {
		ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();
		Task task = new Task();
		System.out.println("main: executing the task");
		Future<String> result = executor.submit(task);
		TimeUnit.SECONDS.sleep(2);
		System.out.println("main: canceling the task");
		result.cancel(true);
		System.out.println("main: cancelled:" + result.isCancelled());
		System.out.println("main: Done:" + result.isDone());
		executor.shutdown();
		System.out.println("main: the executor has finished");
	}
}
```

线程有时候会因为异常导致中断，异常发生后一定要处理异常中断导致未释放的资源。实现UncaughtExceptionHandler接口可以设置一个异常处理器。

### 添加执行器FutureTask完成后的后续处理

有时，我们在处理完成任务之后需要进行后续处理，例如清理资源，发送报表。FutureTask的done方法可以实现一些后续的处理。

执行任务的类需要放到FutureTask中，FutureTask会直接控制任务的执行，任务执行以后，会调用done方法的实现。

```java
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

public class ExecutableTask implements Callable<String> {
	private String name;

	public String getName() {
		return name;
	}

	public ExecutableTask(String name) {
		this.name = name;
	}

	@Override
	public String call() throws Exception {
		long duration = (long) (Math.random() * 10);
		System.out.println(name + ":waiting " + duration + " seconds for results");
		TimeUnit.SECONDS.sleep(duration);
		return "hello, world. i'm " + name;
	}
}
```

```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

public class ResultTask extends FutureTask<String> {
	private String name;

	public ResultTask(Callable<String> callable) {
		super(callable);
		name = ((ExecutableTask) callable).getName();
	}

	@Override
	protected void done() {
		if (isCancelled()) {
			System.out.println(name + ": has been canceled");
		} else {
			System.out.println(name + ": has finished");
		}
	}
}
```

```java
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) throws InterruptedException, ExecutionException {
		ExecutorService executor = Executors.newCachedThreadPool();
		ResultTask[] resultTasks = new ResultTask[5];
		for (int i = 0; i < 5; i++) {
			ExecutableTask executableTask = new ExecutableTask("task " + i);
			resultTasks[i] = new ResultTask(executableTask);
			executor.submit(resultTasks[i]);
		}
		TimeUnit.SECONDS.sleep(5);
		for (int i = 0; i < resultTasks.length; i++) {
			boolean isCancle = resultTasks[i].cancel(true);
			System.out.println("task " + i + " cancel :" + isCancle);
		}
		for (int i = 0; i < resultTasks.length; i++) {
			if (!resultTasks[i].isCancelled()) {
				System.out.println(resultTasks[i].get());
			}
		}
		executor.shutdown();
	}
}
```

执行完成后，可以获得任务执行的结果。

### CompletionService分离任务启动与结果处理
我们有一批报表需要生成是，生成之后要将报表发送出去。

CompletionService可以通过submit方法执行任务。我们可以创建一个任务，这个任务中有一个CompletionService对象处理报表生成。

```java
import java.util.concurrent.CompletionService;

public class ReportRequest implements Runnable {
	private String name;
	private CompletionService<String> service;

	public ReportRequest(String name, CompletionService<String> service) {
		this.name = name;
		this.service = service;
	}

	@Override
	public void run() {
		ReportGenerator reportGenerator = new ReportGenerator(name, "report");
		service.submit(reportGenerator);
	}
}
```

用来生成报表的。

```java
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

public class ReportGenerator implements Callable<String> {
	private String sender;
	private String title;

	public ReportGenerator(String sender, String title) {
		this.sender = sender;
		this.title = title;
	}

	@Override
	public String call() throws Exception {
		long duration = (long) (Math.random() * 10);
		System.out.println(sender + "_" + title + ":reportGenerator: generating a report during " + duration + " seconds");
		TimeUnit.SECONDS.sleep(duration);
		String ret = sender + ": " + title;
		return ret;
	}
}
```

将之前的CompletionService对象的处理结果取出来，进行下一步的处理。

```java
import java.util.concurrent.CompletionService;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

public class ReportProcessor implements Runnable {
	private CompletionService<String> service;
	private boolean end;

	public ReportProcessor(CompletionService<String> service) {
		this.service = service;
		end = false;
	}

	@Override
	public void run() {
		while (!end) {
			try {
				Future<String> result = service.poll(20, TimeUnit.SECONDS);
				if (result != null) {
					String report = result.get();
					System.out.println("reportReceiver:report received:" + report);
				}
			} catch (Exception e) {
				e.printStackTrace();
			}
			System.out.println("reportSender: End");
		}
	}

	public void setEnd(boolean end) {
		this.end = end;
	}
}
```

创建一个CompletionService对象，用来处理报表生成，并保存报表生成后的结果。

```java
import java.util.concurrent.CompletionService;
import java.util.concurrent.ExecutorCompletionService;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Main {
	public static void main(String[] args) throws InterruptedException {
		ExecutorService executor = Executors.newCachedThreadPool();
		CompletionService<String> service = new ExecutorCompletionService<String>(executor);
		ReportRequest faceRequest = new ReportRequest("Face", service);
		ReportRequest onlineRequest = new ReportRequest("Online", service);
		Thread faceThread = new Thread(faceRequest);
		Thread onlineThread = new Thread(onlineRequest);
		ReportProcessor processor = new ReportProcessor(service);
		Thread senderThread = new Thread(processor);
		System.out.println("main: starting the threads");
		faceThread.start();
		onlineThread.start();
		senderThread.start();
		System.out.println("main: waiting for the report generators.");
		faceThread.join();
		onlineThread.join();
		System.out.println("main: shutting down the executor.");
		executor.shutdown();
		executor.awaitTermination(1, TimeUnit.DAYS);
		processor.setEnd(true);
		System.out.println("main: end");
	}
}
```

### 处理被执行器拒绝的任务
有时，由于处理任务已关闭，但是任务队列中还有很多任务没有处理，或者某些任务因为已达到某些条件条件，拒绝执行任务。我们需要处理这些被拒绝执行的任务。

ThreadPoolExecutor有一个方法setRejectedExecutionHandler，接收一个实现RejectedExecutionHandler接口的对象，执行器有被拒绝执行的任务时会默认处理这些被拒绝执行的任务。

实现RejectedExecutionHandler接口，添加自己的处理逻辑。

```java
import java.util.concurrent.RejectedExecutionHandler;
import java.util.concurrent.ThreadPoolExecutor;

public class RejectedTaskController implements RejectedExecutionHandler {

	@Override
	public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
		System.out.println("rejectedTaskController: The task " + r.toString() + " has been rejected");
		System.out.println("rejectedTaskController: " + executor.toString());
		System.out.println("rejectedTaskController: Terminating: " + executor.isTerminating());
		System.out.println("rejectedTaskController: Terminated: " + executor.isTerminated());
	}
}
```

创建一个正常处理的任务。

```java
import java.util.concurrent.TimeUnit;

public class Task implements Runnable {
	private String name;

	public Task(String name) {
		this.name = name;
	}

	@Override
	public void run() {
		System.out.println("Task " + name + ": starting");
		long duration = (long) (Math.random() * 10);
		System.out.println("task " + name + ": reportGenerator: generating a report during " + duration + "seconds");
		try {
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("task " + name + ": Ending");
	}

	@Override
	public String toString() {
		return name;
	}
}
```

我们给ThreadPoolExecutor对象的setRejectedExecutionHandler赋值后，会自动执行RejectedExecutionHandler对象的rejectedExecution方法。

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

public class Main {
	public static void main(String[] args) {
		RejectedTaskController controller = new RejectedTaskController();
		ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();
		executor.setRejectedExecutionHandler(controller);
		System.out.println("main: starting.");
		for (int i = 0; i < 3; i++) {
			Task task = new Task("task " + i);
			executor.submit(task);
		}
		System.out.println("main: shutting down the executor.");
		executor.shutdown();
		System.out.println("main: sending another task.");
		Task task = new Task("rejectedTask");
		executor.submit(task);
		System.out.println("main : end");
	}
}
```

### 任务取消
在Java中，没有哪一种用来停止线程的方法是绝对安全的，因此没有哪一种方法优先用来停止任务。

中断：线程中断是一个协作机制，一个线程给另一个线程发送信号（signal），通知它在方便或者可能的情况下通知正在做的工作，去做其他事情。

在API和语言规范中，并没有把中断与任何取消的语意绑定起来，但是，实际上，使用中断来处理取消之外的任何事情都是不明智的，并且很难支撑起更大的应用。

调用interrupt并不意味着必然停止目标线程正在进行的工作：它仅仅传递了请求中断的消息。

静态的 interrupted 应该小心使用，因为它会清除并发线程的中断状态。如果你调用了 interrupted，并且它返回了 true，你必须对其进行处理，除非你想掩盖这个中断——你可以抛出 InterruptedException，或者通过再次调用 interrupt 来保存中断状态。

中断通常是实现取消最明智的选择。