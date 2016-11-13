---
date: 2016-10-31 22:16:12+08:00
layout: post
title: 一起学java并发编程（十三）：Fork/Join框架的实践
thread: 15
categories: java
tags: java并发编程
---
Java7引入的Fork/Join框架，通过分治技术将问题拆分成小任务。在一个任务中，先检查将要解决的问题的大小，如果大于一个设定的大小，那就将问题拆分成可以通过框架来执行的小任务。如果问题的大小小于设定的大小，就直接在任务里解决这个问题。

Join操作会让主任务等待它所创建的子任务的完成。执行任务的线程称为工作者线程。工作者线程寻找其他未执行的任务，然后开始执行。
因为这种任务执行特性，Fork/Join有以下限制：

- 任务只能使用fork()和join()操作当作同步机制。如果使用其他的同步机制，工作者线程就不能执行其他任务，当然这些任务是在同步操作里时。比如，如果在Fork/Join框架中将一个任务休眠，正在执行这个任务的工作者线程在休眠期内不能执行另一个任务。
- 任务不能执行I/O操作，比如文件数据的读取与写入。
- 任务不能抛出非运行时异常，必须在代码中处理掉这些异常。

Fork/Join框架的核心是由下列两个类组成的：

- ForkJoinPool：这个类实现了ExecutorService接口和工作窃取算法。它管理工作者线程，并提供任务的状态信息，以及任务的执行信息。
- ForkJoinTask：这个类是一个将在ForkJoinPool中执行的任务的基类。

为了实现Fork/Join任务，需要实现以下两个类之一：

- RecursiveAction：用于任务没有返回结果的场景。
- RecursiveTask：用于任务有返回结果的场景。

### Fork/Join的使用
我们定义一个产品类，产品有名称和价格两个属性。

```java
public class Product {
	private String name;
	private double price;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public double getPrice() {
		return price;
	}

	public void setPrice(double price) {
		this.price = price;
	}
}
```

在定义一个生产商品的类，生成的商品的名称不同，价格都是10。

```java
public class ProductListGenerator {
	public List<Product> generator(int size) {
		List<Product> ret = new ArrayList<Product>();
		for (int i = 0; i < size; i++) {
			Product product = new Product();
			product.setName("product " + i);
			product.setPrice(10);
			ret.add(product);
		}
		return ret;
	}
}
```

然后定义任务执行的类。这里我们不需要任务执行的结果，所以选择继承RecursiveAction。

```java
public class Task extends RecursiveAction {
	private static final long serialVersionUID = 1L;
	private List<Product> products;
	private int first;
	private int last;
	private double increment;

	public Task(List<Product> products, int first, int last, double increment) {
		this.products = products;
		this.first = first;
		this.last = last;
		this.increment = increment;
	}

	@Override
	protected void compute() {
		if (last - first < 10) {
			updatePrices();
		} else {
			int middle = (last + first) / 2;
			System.out.println("task: pending tasks:" + getQueuedTaskCount());
			Task t1 = new Task(products, first, middle + 1, increment);
			Task t2 = new Task(products, middle + 1, last, increment);
			invokeAll(t1, t2);
		}
	}

	private void updatePrices() {
		for (int i = first; i < last; i++) {
			Product product = products.get(i);
			product.setPrice(product.getPrice() * (1 + increment));
		}
	}
}
```

这里的任务负责商品的价格更新，一次只负责更新不超过10个商品。如果一次处理的商品数超过10个，就拆分成两个子任务。

在main方法中，我们可以启动任务，并检查任务的处理结果。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		ProductListGenerator generator = new ProductListGenerator();
		List<Product> products = generator.generator(10000);
		Task task = new Task(products, 0, products.size(), 0.2);
		ForkJoinPool pool = new ForkJoinPool();
		pool.execute(task);
		do {
			System.out.println("main: thread count: " + pool.getActiveThreadCount());
			System.out.println("main: thread steal: " + pool.getStealCount());
			System.out.println("main: parallelism: " + pool.getParallelism());
		} while (!task.isDone());
		pool.shutdown();
		if (task.isCompletedNormally()) {
			System.out.println("main: the process has completed normally.");
		}
		for (int i = 0; i < products.size(); i++) {
			Product product = products.get(i);
			if (product.getPrice() != 12) {
				System.out.println("product " + product.getName() + ": " + product.getPrice());
			}
		}
		System.out.println("main: end of the program.");
	}
}
```

这个例子我们主要关注的是ForkJoinPool对象和RecursiveAction的子类。我们使用了ForkJoinPool的默认配置，ForkJoinPool默认会创建一个线程数与当前CPU数相等的线程池。RecursiveAction的invokeAll()方法负责执行主任务创建的子任务，这是一个同步的调用，主任务会等待它的子任务完成，然后继续执行。当一个主任务等待它的子任务时，执行这个主任务的工作者线程接收另一个等待执行的任务并开始执行。正因为有了这个行为，所以Fork/Join框架提供了一种比Runnable和Callable对象更加高效的任务管理机制。

### Fork/Join合并处理结果
上面示例中执行的任务没有执行结果。如果需要获取任务的执行结果，则需要RecursiveTask类来实现。RecursiveTask继承了ForkJoinTask类，并且实现了由执行器框架（Executor Framework）提供的Future接口。

这种有结果的任务和无结果的任务执行处理方式大致相同，但是多了一个合并处理结果的步骤。下面的示例是从文档中查找一个词，统计这个词出现的次数。

首先，模拟一个文档。

```java
public class Document {
	private String[] words = { "the", "hello", "goodbye", "packt", "java", "thread", "pool", "random", "class", "main" };

	public String[][] generateDocument(int numLines, int numWords, String word) {
		int counter = 0;
		String[][] document = new String[numLines][numWords];
		Random random = new Random();
		for (int i = 0; i < numLines; i++) {
			for (int j = 0; j < numWords; j++) {
				int index = random.nextInt(words.length);
				document[i][j] = words[index];
				if (document[i][j].equals(word)) {
					counter++;
				}
			}
		}
		System.out.println("document: the word appears " + counter + " times in the document");
		return document;
	}
}
```

创建统计这个文档有多少行的任务类，继承RecursiveTask。

```java
public class DocumentTask extends RecursiveTask<Integer> {

	private static final long serialVersionUID = 1L;
	private String[][] document;
	private int start, end;
	private String word;

	public DocumentTask(String[][] document, int start, int end, String word) {
		this.document = document;
		this.start = start;
		this.end = end;
		this.word = word;
	}

	@Override
	protected Integer compute() {
		int result = 0;
		if (end - start < 10) {
			result = processLines(document, start, end, word);
		} else {
			int mid = (start + end) / 2;
			DocumentTask task1 = new DocumentTask(document, start, mid, word);
			DocumentTask task2 = new DocumentTask(document, mid, end, word);
			invokeAll(task1, task2);
			try {
				result = groupResults(task1.get(), task2.get());
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		return result;
	}

	private int processLines(String[][] document2, int start2, int end2, String word2) {
		List<LineTask> tasks = new ArrayList<LineTask>();
		for (int i = start; i < end; i++) {
			LineTask task = new LineTask(document[i], 0, document[i].length, word);
			tasks.add(task);
		}
		invokeAll(tasks);
		int result = 0;
		for (int i = 0; i < tasks.size(); i++) {
			LineTask task = tasks.get(i);
			try {
				result = result + task.get();
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		return result;
	}

	private int groupResults(Integer integer, Integer integer2) {
		return integer + integer2;
	}
}
```

创建统计一行中某个词出现次数的任务类，目的是统计某个词在每行中出现的次数，将统计结果交给DocumentTask。也是继承RecursiveTask。

```java
public class LineTask extends RecursiveTask<Integer> {
	private static final long serialVersionUID = 1L;

	private String[] line;
	private int start, end;
	private String word;

	public LineTask(String[] strings, int start, int end, String word) {
		this.line = strings;
		this.start = start;
		this.end = end;
		this.word = word;
	}

	@Override
	protected Integer compute() {
		Integer result = null;
		if (end - start < 100) {
			result = count(line, start, end, word);
		} else {
			int mid = (start + end) / 2;
			LineTask task1 = new LineTask(line, start, mid, word);
			LineTask task2 = new LineTask(line, mid, end, word);
			invokeAll(task1, task2);
			try {
				result = groupResults(task1.get(), task2.get());
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		return result;
	}

	private Integer count(String[] line, int start, int end, String word) {
		int counter = 0;
		for (int i = start; i < end; i++) {
			if (line[i].equals(word)) {
				counter++;
			}
		}
		return counter;
	}

	private Integer groupResults(Integer integer, Integer integer2) {
		return integer + integer2;
	}
}
```

最后我们在Main类中模拟一个文档，并监控任务的执行。

```java
public class Main {
	public static void main(String[] args) throws Exception {
		Document doc = new Document();
		String[][] document = doc.generateDocument(100, 1000, "the");
		DocumentTask task = new DocumentTask(document, 0, 100, "the");
		ForkJoinPool pool = new ForkJoinPool();
		pool.execute(task);
		do {
			System.out.println("********************");
			System.out.println("main: parallelism: " + pool.getParallelism());
			System.out.println("main: active threads: " + pool.getActiveThreadCount());
			System.out.println("main: task count: " + pool.getQueuedTaskCount());
			System.out.println("main: steal count: " + pool.getStealCount());
		} while (!task.isDone());
		pool.shutdown();
		pool.awaitTermination(1, TimeUnit.DAYS);
		System.out.println("main: the word appears " + task.get() + " in the document");
	}
}
```

这个例子中，DocumentTask的作用是将一个超过10行的文档拆成两个小文档，直到处理的文档的行数小于10行。LineTask的作用是将一个词数超过100的行拆成两个小任务，直到每个任务处理的词数小于100。groupResults()方法是有返回任务结果与没有返回任务结果的Fork/Join执行器主要不同之处。

### Fork/Join异步运行任务
Fork/Join框架执行任务的时候可以采用同步或异步的方式。当采用同步方式执行时，直到任务完成，线程才返回结果。而采用异步方式执行时，线程会立即返回结果，任务继续执行。因此，采用同步线程执行的方式，允许ForkJoinPool类采用工作窃取算法来分配一个任务给在执行休眠任务的工作者线程。相反，当采用异步执行方法时，ForkJoinPool类无法使用工作窃取算法来提升应用程序的性能。

下面的示例中，只有调用join()或get()方法来等待任务的结束时，ForkJoinPool类才可以使用工作窃取算法。

定义一个任务，用来查找指定扩展名的文件，当找到的是一个目录时，就新建一个任务，否则就是文件，判断文件的后缀名。

```java
public class FolderProcessor extends RecursiveTask<List<String>> {

	private static final long serialVersionUID = 1L;
	private String path;
	private String extension;

	public FolderProcessor(String path, String extension) {
		this.path = path;
		this.extension = extension;
	}

	@Override
	protected List<String> compute() {
		List<String> list = new ArrayList<String>();
		List<FolderProcessor> tasks = new ArrayList<FolderProcessor>();
		File file = new File(path);
		File[] content = file.listFiles();
		if (content != null) {
			for (int i = 0; i < content.length; i++) {
				if (content[i].isDirectory()) {
					FolderProcessor task = new FolderProcessor(content[i].getAbsolutePath(), extension);
					task.fork();
					tasks.add(task);
				} else if (checkFile(content[i].getName())) {
					list.add(content[i].getAbsolutePath());
				}
			}
		}
		if (tasks.size() > 50) {
			System.out.println(file.getAbsolutePath() + ": " + tasks.size() + " task ran.");
		}
		addResultsFromTasks(list, tasks);
		return list;
	}

	private void addResultsFromTasks(List<String> list, List<FolderProcessor> tasks) {
		for (FolderProcessor folderProcessor : tasks) {
			list.addAll(folderProcessor.join());
		}

	}

	private boolean checkFile(String name) {
		return name.endsWith(extension);
	}
}
```

执行任务，并打印执行结果。

```java
public class Main {
	public static void main(String[] args) {
		ForkJoinPool pool = new ForkJoinPool();
		FolderProcessor system = new FolderProcessor("C:\\Windows", "log");
		FolderProcessor app = new FolderProcessor("C:\\Program Files", "log");
		FolderProcessor documents = new FolderProcessor("C:\\Documents And Settings", "log");
		pool.execute(system);
		pool.execute(app);
		pool.execute(documents);
		do {
			System.out.println("**************");
			System.out.println("main: parallelism :" + pool.getParallelism());
			System.out.println("main: active threads:" + pool.getActiveThreadCount());
			System.out.println("main: tasks count:" + pool.getQueuedTaskCount());
			System.out.println("main: steal count:" + pool.getStealCount());
		} while (!system.isDone() || !app.isDone() || !documents.isDone());
		pool.shutdown();
		List<String> results = system.join();
		System.out.println("system: " + results.size() + " files founds.");
		results = app.join();
		System.out.println("apps: " + results.size() + " files founds.");
		results = documents.join();
		System.out.println("documents: " + results.size() + " files founds.");
	}
}
```

这里每个RecursiveTask调用fork()方法把新新任务放到线程池中。fork()方法发送任务到线程池时，如果线程池中有空闲的工作者线程或者将创建一个新的线程，那么开始执行这个任务，fork()方法会立即返回。此时，主线程可以继续处理文件夹里的其他内容。

一旦主线程处理完指定文件夹里的所有内容，它将调用join()方法等待发送到线程池中的所有子任务执行完成。

### Fork/Join异常处理
与其他的线程执行框架一样，ForkJoinPool也有异常处理方式。不能在ForkJoinTask类的compute()方法中抛出任务非运行时异常，因为这个方法的实现没有包含任务throws声明。因此，需要包含必需的代码来处理相关的异常。在控制台上，程序没有结束执行，不能看到任务异常信息。如果异常不能被抛出，那么它只是简单地将异常吞噬掉。我们可以利用ForkJoinTask的一些方法获知任务是否有异常抛出，以及抛出的异常类型。

我们模拟一个异常的处理过程。

```java
public class Task extends RecursiveTask<Integer> {
	private static final long serialVersionUID = 1L;
	private int[] array;
	private int start, end;

	public Task(int[] array, int start, int end) {
		this.array = array;
		this.start = start;
		this.end = end;
	}

	@Override
	protected Integer compute() {
		System.out.println("task: start from " + start + " to " + end);
		if (end - start < 10) {
			if (3 > start && 3 < end) {
				throw new RuntimeException("this task throws an exception: task from " + start + " to " + end);
			}
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		} else {
			int mid = (end + start) / 2;
			Task task1 = new Task(array, start, mid);
			Task task2 = new Task(array, mid, end);
			invokeAll(task1, task2);
		}
		System.out.println("task: end form " + start + " to " + end);
		return 0;
	}
}
```

我们能够通过RecursiveTask的isCompletedAbnormally判断是否有异常抛出，并能够通过getException()方法获得抛出的异常。

### Fork/Join取消任务
在ForkJoinPool类中执行ForkJoinTask对象时，在任务开始执行前可以取消它。ForkJoinTask类提供了cancel()方法来达到取消任务的目的。在取消一个任务时必须要注意一下两点：

- ForkJoinPool类不提供任何方法来取消线程池中正在运行或者等待运行的所有任务；
- 取消任务时，不能取消已经被执行的任务。

我们实现一个取消ForkJoinTask对象的范例。该范例查找数组中某个数字所处的位置。

创建一个生成随机数组的类。

```java
public class ArrayGenerator {
	public int[] generateArray(int size) {
		int[] array = new int[size];
		Random random = new Random();
		for (int i = 0; i < size; i++) {
			array[i] = random.nextInt(10);
		}
		return array;
	}
}
```

创建一个任务管理类，用来管理任务的新增和取消。

```java
public class TaskManager {
	private List<ForkJoinTask<Integer>> tasks;

	public TaskManager() {
		tasks = new ArrayList<ForkJoinTask<Integer>>();
	}

	public void addTask(ForkJoinTask<Integer> task) {
		tasks.add(task);
	}

	public void cancelTasks(ForkJoinTask<Integer> cancelTask) {
		for (ForkJoinTask<Integer> forkJoinTask : tasks) {
			if (forkJoinTask.equals(cancelTask)) {
				forkJoinTask.cancel(true);
				((SearchNumberTask) forkJoinTask).writeCancelMessage();
			}
		}
	}
}
```

继承并实现RecursiveTask类，完成数字搜索。

```java
public class SearchNumberTask extends RecursiveTask<Integer> {

	private static final long serialVersionUID = 1L;
	private int[] numbers;
	private int start, end;
	private int number;
	private TaskManager manager;
	private final static int NOT_FOUND = -1;

	public SearchNumberTask(int[] numbers, int start, int end, int number, TaskManager manager) {
		this.numbers = numbers;
		this.start = start;
		this.end = end;
		this.number = number;
		this.manager = manager;
	}

	@Override
	protected Integer compute() {
		System.out.println("task: " + start + " : " + end);
		int ret;
		if (end - start > 10) {
			ret = launchTasks();
		} else {
			ret = lookForNumber();
		}
		return ret;
	}

	private int lookForNumber() {
		for (int i = 0; i < end; i++) {
			if (numbers[i] == number) {
				System.out.println("task: number " + number + " found in position " + i);
				manager.cancelTasks(this);
				return i;
			}
		}
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return NOT_FOUND;
	}

	private int launchTasks() {
		int mid = (start + end) / 2;
		SearchNumberTask task1 = new SearchNumberTask(numbers, start, mid, number, manager);
		SearchNumberTask task2 = new SearchNumberTask(numbers, mid, end, number, manager);
		manager.addTask(task1);
		manager.addTask(task2);
		task1.fork();
		task2.fork();
		int returnValue = task1.join();
		if (returnValue != -1) {
			return returnValue;
		}
		returnValue = task2.join();
		return returnValue;
	}

	public void writeCancelMessage() {
		System.out.println("task: cancelled task from " + start + " to " + end);
	}

}
```

执行并查看任务取消的过程。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		ArrayGenerator generator = new ArrayGenerator();
		int[] array = generator.generateArray(1000);
		TaskManager manager = new TaskManager();
		ForkJoinPool pool = new ForkJoinPool();
		SearchNumberTask task = new SearchNumberTask(array, 0, 1000, 5, manager);
		pool.execute(task);
		pool.shutdown();
		pool.awaitTermination(1, TimeUnit.DAYS);
		System.out.println("main: the program has finished");
	}
}
```

ForkJoinTask类提供的cancel()方法允许取消一个仍没有被执行的任务，这是非常重要的一点。如果任务已经开始执行，那么调用cancel()也无法取消。这个方法接受一个名为mayInterruptRunning的boolean值参数。如果传递true值给这个方法，即使任务正在运行也会被取消。JavaAPI文档指出，这个值默认是没有任何作用的，因为中断不能用来控制取消。

总之，Fork/Join任务执行框架为多线程执行新增了一种高效的实现。