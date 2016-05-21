---
date: 2016-05-21 11:49:12+08:00
layout: post
title: Quartz开发指南（翻译）
thread: 2
categories: 翻译
tags: quartz
---

#Quartz调度器开发指南
V 2.2.1

###目录
####[使用Quartz API](#1)
######[实例化Scheduler](#1.1)
######[关键接口](#1.2)
######[Job和Trigger](#1.3)
####[使用Job和JobDetail](#2)
######[Job和JobDetail](#2.1)
####[使用trigger](#3)
######[Trigger](#3.1)
######[SimpleTrigger](#3.2)
######[CronTriggers](#3.3)
####[使用TriggerListener和JobListener](#4)
######[TriggerListener和JobListener](#4.1)
######[创建你自己的监听器](#4.2)
####[使用SchedulerListener](#5)
######[SchedulerListener](#5.1)
######[添加一个SchedulerListener](#5.2)
######[移除一个SchedulerListener](#5.3)
####[使用JobStore](#6)
######[关于JobStores](#6.1)
######[RAMJobStore](#6.2)
######[JDBCJobStore](#6.3)
######[TerracottaJobStore](#6.4)
####[配置，Scheduler Factory和日志](#7)
######[配置组件](#7.1)
######[Scheduler工厂](#7.2)
######[日志](#7.3)
####[Quartz调度器的其他特性](#8)
######[插件](#8.1)
######[JobFactory](#8.2)
######[Factory自带Job](#8.3)
####[高级功能](#9)
######[集群](#9.1)
######[JTA事务](#9.2)

<h2 id="1">使用Quartz API</h2>
<h4 id="1.1">实例化Scheduler</h4>
<p>在你使用Scheduler之前，首先需要实例化。你必须使用SchedulerFactory来完成。</p>
<p>很多Quartz的使用者在JNDI中存储一个工厂的实例，这样非常容易直接使用一个工厂实例。</p>
<p>一旦一个scheduler被实例化，它就能被启动、设置为备用模式、停止。请注意，一旦你关闭了scheduler，不重新实例化就无法重启它。Scheduler不启动trigger也不会启动（因此，job也不会执行）。Scheduler处于暂停状态它们都不会工作。</p>
<p>下面的代码段实例化并启动一个scheduler，安排一个job执行。</p>
```java
SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();
Scheduler sched = schedFact.getScheduler();
sched.start();
// define the job and tie it to our HelloJob class
JobDetail job = new Job(HelloJob.class)
		.withIdentity("myJob", "group1")
		.build();
// Trigger the job to run now, and then every 40 seconds
Trigger trigger = new Trigger()
		.withIdentity("myTrigger", "group1")
		.startNow()
		.withSchedule(simpleSchedule().withIntervalInSeconds(40).repeatForever())
		.build();
// Tell quartz to schedule the job using our trigger
sched.scheduleJob(job, trigger);
```
<h4 id="1.2">关键接口</h4>
Quartz API的关键接口：

- Scheduler——Scheduler进行交互的主要API。
- Job——你想让Scheduler执行时需要实现的接口组件。
- JobDetail——用于定义Job的实例。
- Trigger——一个定义在将要执行Job的scheduler上的组件。
- JobBuilder——已定义的Job实例，用于定义/构建的JobDetail实例。
- TriggerBuilder——用于定义/构建Trigger实例。

<p>一个scheduler的生命周期是有限的，从SchedulerFactory开始直到调用它的shutdown()方法。</p>
<p>Scheduler接口一旦创建，就能添加、移除、和列出job和trigger，并执行另一些与scheduling相关的操作(例如暂停trigger)，然而，在scheduler被start()方法启动前，它不会作用于任何trigger（执行job），见第3页“实例化Scheduler”。</p>
<p>Quartz提供”builder”类定义领域特定语言(或DSL，有时也被称为“连贯接口”)。这里有一个例子：</p>
```java
		// define the job and tie it to our HelloJob class
		JobDetail job = new Job(HelloJob.class)
				.withIdentity("myJob", "group1") // name "myJob", group "group1"
				.build();
		// Trigger the job to run now, and then every 40 seconds
		Trigger trigger = new Trigger()
				.withIdentity("myTrigger", "group1")
				.startNow()
				.withSchedule(simpleSchedule().withIntervalInSeconds(40).repeatForever())
				.build();
		// Tell quartz to schedule the job using our trigger
		sched.scheduleJob(job, trigger);
```
<p>使用JobBuilder类的静态方法创建job的代码块。同样，构建trigger的代码块使用TriggerBuilder类的静态方法创建，SimpleScheduleBulder类也一样。</p>
<p>DSL的静态导入可以接受这样的导入语句：</p>
```java
import static org.quartz.JobBuilder.*;
import static org.quartz.SimpleScheduleBuilder.*;
import static org.quartz.CronScheduleBuilder.*;
import static org.quartz.CalendarIntervalScheduleBuilder.*;
import static org.quartz.TriggerBuilder.*;
import static org.quartz.DateBuilder.*;
```
<p>不同的SchedulerBuilder类拥有不同定义schedules类型的方法。</p>
<p>DateBuilder类包含有不同类型的方法去简单的构造java.util.Date实例的特定的时间点（例如如果当前时间为9:43:27，下一小时执行的时间，或者换句话说10:00）。</p>
<h4 id="1.3">Job和Trigger</h4>
<p>一个job就是实现了Job接口的类。如下所示，这个接口有一个简单的方法：</p>
```java
package org.quartz;
public interface Job {
			public void execute(JobExecutionContext context) throws JobExecutionException;
}
```
<p>当一个job的触发器触发时，execute(..)方法被Scheduler的一个工作线程调用。JobExecutionContext对象就是通过这个方法提供job实例的信息，如运行环境，包括scheduler执行的句柄，trigger触发执行的句柄，job的JobDetail对象，和一些其它信息。</p>
<p>JobDetail对象由Quartz客户端（你的程序）在job添加进scheduler的时候创建。这个对象包含了很多job的属性设置，和JobDataMap一样，它能够存储你的job实例的状态信息。JobDetail对象本质上是定义的job实例。</p>
<p>Trigger对象常常用于触发job的执行（或者触发），当你希望安排一个job，你可以实例化一个trigger，并“调整”其属性来提供你想要的调度。</p>
<p>Trigger可能有一个与它们关联的JobDataMap对象。JobDataMap用于传递特定于触发器触发的工作参数。Quartz附带一些不同的触发类型，但是最常用的就是SimpleTrigger和CronTrigger。</p>

- SimpleTrigger很方便，如果你需要“一次性”执行(只是在一个给定时刻执行job),或者如果你需要一个job在一个给定的时间,并让它重复N次,并在执行之间延迟T。
- CronTrigger是有用的，如果你想拥有引发基于当前日历时间表，如“每个星期五,中午”或“在每个月的第十天 10:15。”

<p>为什么会同时有job和trigger？许多工作调度器没有单独的job和trigger的概念。一些定义一个job只是一个执行时间（或计划）以及一些小的工作标识符。其它的大部分就像Quartz的job和triiger对象的结合体。Quartz旨在创建一个分离的scheduler和scheduler的执行。这个设计有很多好处。</p>
<p>例如，你可以创建独立于trigger的job并将他们存储在job scheduler。这使你能在同一个job上关联多个trigger。另一个解耦的好处是，能够在scheduler关联的trigger已过期时仍能配置job。这使你在重新安排它们后，不用重新定义它们。它还允许您修改或替换一个trigger，而不必重新定义相关的job。</p>
#####特性
<p>当Job和trigger注册进Quartz scheduler时给出了key。Job和trigger（JobKey和TriggerKey）的key允许它们放被放置到group中，group可用于分类组织你得job和trigger，例如“报表job”和“维护job”。Job或者trigger的key值在group中必须是唯一的。换句话说，job或trigger的完整的key(或标识)是由key和group组成。</p>
<p>有关job和trigger的详细信息,请参见第6页“Job和JobDetails”和第10页“使用trigger”。</p>
 
<h2 id="2">使用Job和JobDetail</h2>
<h4 id="2.1">Job和JobDetail</h4>
<p>Job很容易实现，接口中只有一个“execute”方法。Job有更多的东西你需要了解，就是关于Job接口的execute(..)方法，和JobDetails。</p>
<p>当你实现一个job类时，代码知道如何实现job的特定类型的实际工作，你或许希望了解job实例拥有的Quartz的各种属性。</p>
<p>使用JobBuilder类构建JobDetail实例。你通常会想要使用静态导入导入所有的方法,为了你的代码能够DSL-feel。</p>
```java
import static org.quartz.JobBuilder.*;
```
<p>下面的代码片段定义了一个job并安排它的执行：</p>
```java
		// define the job and tie it to our HelloJob class
		JobDetail job = new Job(HelloJob.class)
				.withIdentity("myJob", "group1") // name "myJob", group "group1"
				.build();
		// Trigger the job to run now, and then every 40 seconds
		Trigger trigger = new Trigger()
				.withIdentity("myTrigger", "group1")
				.startNow()
				.withSchedule(simpleSchedule().withIntervalInSeconds(40).repeatForever())
				.build();
		// Tell quartz to schedule the job using our trigger
		sched.scheduleJob(job, trigger);
```
<p>现在考虑这个job类HelloJob，如下所示:</p>
```java
public class HelloJob implements Job {
	public HelloJob() {
	}

	public void execute(JobExecutionContext context) throws JobExecutionException {
		System.err.println("Hello! HelloJob is executing.");
	}
}
```
<p>请注意我们给scheduler一个JobDetail实例，而且它知道要执行的job的类型，只需提供我们构建JobDetail的job类。每次scheduler执行job,它会在调用它的execute(..)方法之前创建这个类的新实例。当执行完成，job实例的引用被删除,然后垃圾收集器回收实例。</p>
<p>这种行为的后果之一是job必须有一个无参数的构造函数(当使用默认JobFactory实现)。另一个是job类上定义的没有意义的数据字段的状态,它们的值在Job执行期间不会被保存。</p>
<p>你能够使用JobDataMap为job实例提供属性/配置或跟踪job执行间的状态,,它是JobDetail对象的一部分。</p>
#####JobDataMap
<p>JobDataMap可以用来保存大量那些在job实例执行时你希望得到的 (序列化)数据对象。JobDataMap是Java Map接口的一个实现,并且添加一些便利的方法来存储和检索数据的原始类型。</p>
<p>当定义/构建JobDetail，这里有一些将job添加进scheduler之前放到JobDataMap的数据。</p>
```java
		// define the job and tie it to our DumbJob class
		JobDetail job = new Job(DumbJob.class)
				.withIdentity("myJob", "group1") // name "myJob", group "group1"
				.usingJobData("jobSays", "Hello World!")
				.usingJobData("myFloatValue", 3.141f)
				.build();
```
<p>这是一个例子，在job执行期间，从JobDataMap获取数据:</p>
```java
public class DumbJob implements Job {
	public DumbJob() {
	}

	public void execute(JobExecutionContext context) throws JobExecutionException {
		JobKey key = context.getJobDetail().getKey();
		JobDataMap dataMap = context.getJobDetail().getJobDataMap();
		String jobSays = dataMap.getString("jobSays");
		float myFloatValue = dataMap.getFloat("myFloatValue");
		System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
	}
}
```
<p>如果你使用一个持久化的JobStore(在本教程的JobStore节讨论)你应该注意在决定你在JobDataMap中安排什么,因为里面的对象将序列化,他们因此会有类版本问题。显然标准Java类型应该很安全,但除此之外,任何时候有人改变一个类的定义的序列化实例,必须小心不要打破兼容性。可选地,您可以将JDBC-JobStore和JobDataMap放入model中,只允许基本类型和字符串存储在map中,从而消除任何后来序列化问题的可能性。</p>
<p>如果你为job类添加setter方法,相当于在JobDataMap中对应的key (如上面的示例中setJobSays(String val)方法，然后quartz的默认JobFactory实现将自动调用这些setter当Job被实例化,从而防止需要显式地在你的执行方法获得map的值。</p>
<p>trigger也可以有与他们有关的JobDataMaps。如果你有一个job存储在scheduler由多个trigger常规重复使用,然而每个独立的触发,你想工作提供不同的数据输入，这是有用的。</p>
<p>job执行期间在JobExecutionContext上建立JobDataMap十分方便。它是基于JobDetail的JobDataMap和基于Trigger上的一个合并，后者任何相同名字的值将覆盖前者。</p>
<p>这是一个简单的例子在job执行期间从JobExecutionContext合并的JobDataMap获取数据:</p>
```java
public class DumbJob implements Job {
	public DumbJob() {
	}

	public void execute(JobExecutionContext context) throws JobExecutionException {
		JobKey key = context.getJobDetail().getKey();
		JobDataMap dataMap = context.getMergedJobDataMap();
		// Note the difference from the previous example
		String jobSays = dataMap.getString("jobSays");
		float myFloatValue = dataMap.getFloat("myFloatValue");
		ArrayList state = (ArrayList) dataMap.get("myStateData");
		state.add(new Date());
		System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
	}
}
```
<p>或者如果你想依靠JobFactory注入数据映射到类的值,它可能看起来像这样:</p>
```java
public class DumbJob implements Job {
	String jobSays;
	float myFloatValue;
	ArrayList state;

	public DumbJob() {
	}

	public void execute(JobExecutionContext context) throws JobExecutionException {
		JobKey key = context.getJobDetail().getKey();
		JobDataMap dataMap = context.getMergedJobDataMap();
		// Note the difference from the previous example
		state.add(new Date());
		System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
	}

	public void setJobSays(String jobSays) {
		this.jobSays = jobSays;
	}

	public void setMyFloatValue(float myFloatValue) {
		myFloatValue = myFloatValue;
	}

	public void setState(ArrayList state) {
		state = state;
	}
}
```
<p>你会注意到整个代码的类是长,但是execute()方法中的代码更干净。当然你也可以说,虽然代码比较长,它实际上花了更少的编码,如果程序员用IDE自动生成setter方法,而不必手工从JobDataMap检索值。这是你的选择。</p>
#####Job实例
<p>你可以创建一个单一的工作类,在scheduler中存储被JobDetails创建的多个实例定义——每个都有自己的一组属性和JobDataMap——并将它们添加到调度器。</p>
<p>例如,您可以创建一个类称为SalesReportJob,实现了job接口。这个job也许被编码成预计的参数(通过JobDataMap)发送给指定的基于销售报告的销售人员。然后,他们可能会创建多个Job定义(JobDetails),比如SalesReportForJoe和SalesReportForMike中指定相应的JobDataMaps作为“Joe”和“Mike”各自的输入工作。</p>
<p>当trigger触发,JobDetail(实例定义)被关联加载,Job类通过scheduler中的配置JobFactory实例化。默认JobFactory简单地调用Job类的newInstance(),然后试图调用类中与JobDataMap键的名称相匹配setter方法。您可能想要创建自己的JobFactory实现，诸如应用程序的IoC或DI容器生产/实例化job类。</p>
<p>每个存储的JobDetail作为job定义或JobDetail实例被引用,每个执行的job都是一个job实例或job定义的实例。一般来说,使用术语“工作”时,指的是定义的名称或JobDetail。类实现的工作接口,称为“job类”。</p>
#####Job状态和并发
<p>下面是job状态数据和并发的一些注意点。你可以添加一些注解在你的job类上，在一些方面影响Quartz的行为。</p>
<p>@DisallowConcurrentExecution注释,可以添加到job类上，通知Quartz不要同时执行给定的job定义（引用给定的job类）的多个实例。在上一个例子中，如果SalesReportJob用了这个注解，在给定时间将只能执行SalesReportJob的一个实例，但是它可以同时执行“SalesReportForMike”的一个实例。约束是基于实例的定义（JobDetail），而不是job类的实例。然而，它决定了注解参与到类本身中（在Quartz设计时），因为它对类如何编码产生影响。</p>
<p>@PersistJobDataAfterExecution注释可以添加到job类，通知Quartz在execute()方法成功执行后更新JobDetail的JobDataMap的拷贝，这样相同的job下一次执行时接收到更新的值，而不是原来存储的值。和@DisallowConcurrentExecution注解一样，应用到job定义的实例上，而不是job类的实例，虽然它给job类带来了属性，因为它对类如何编码产生影响。（例如，“有状态性”需要明确理解执行方法的代码）。</p>
<p>如果你使用@PersistJobDataAfterExecution注解,您还应该考虑使用@DisallowConcurrentExecution注解,以避免可能的混乱(竞争条件)的数据被存储在相同的工作的两个实例(JobDetail)并发执行。</p>
#####Job的其他属性
<p>这里有你可以通过JobDetail对象定义job实例的另一些属性的快速总结：</p>
<p>持久性——如果一个job不是持久的, 一旦不再有任何活动的与之关联的trigger它将自动从scheduler删除。换句话说,不持久的job的生命周期依赖于它的trigger的存在。<p>
<p>RequestsRecovery——如果一份工作“请求恢复”,在scheduler强制关闭期间执行(即进程是运行崩溃,或机器关闭)，在scheduler再次启动时会重新执行。在这种情况下,JobExecutionContext.isRecovering()方法将返回true。<p>
#####JobExecutionException
<p>最后,我们需要告诉你Job.execute(..)一些细节。唯一类型的异常(包括RuntimException),你可以从执行方法抛出JobExecutionException。因此,你通常应该使用“rycatch”块包裹执行方法。你也应该花些时间看着JobExecutionException的文档,因为你的job可以使用它提供scheduler作为你想如何处理异常的各种指令。</p>
 
<h2 id="3">使用trigger</h2>
<h4 id="3.1">Trigger</h4>
<p>和job一样,trigger容易处理,但是他们有不同的定制项,在你充分使用Quartz之前需要了解它们。</p>
<p>不同类型的trigger用来满足不同的调度需求。常用的有两种，SimpleTrigger和CronTrigger。关于这些特定触发器的细节在第10页“使用trigger”。</p>
#####通用的trigger属性
<p>除了所有的触发器都有TriggerKey属性作为追踪它们的标识符外，它们还有很多通用的其它属性。当你创建trigger定义时你可以使用TriggerBuilder设置这些通用的属性（如下示例）。</p>
<p>这是通用的触发器类型：<p>

- 属性jobKey是trigger触发将要执行的job的标识。
- startTime是trigger的计划第一次生效的标识。值是一个java.util.Date对象，定义了一个给定的日期时间。对于许多类型的trigger来说，trigger应该在开始时间准确的启动。对于其它类型来说标志着scheduler开始被追踪。这意味着你可以这样存储一个scheduler的trigger，例如一月“每月的第5天执行”，如果startTime属性设置为4月1日，在首次触发前，还要过几个月。
- endTime属性表示trigger的计划不再有效。换句话说，一个“每月5号”的trigger，endTime为7月1号，它上次执行的时间就是6月5号。

<p>其它属性，还有更多的解析，将在下面讨论。</p>
#####优先级
<p>有时，当你有很多触发器（或者你得Quartz线程池有很多工作线程），Quartz也许没有足够的资源在同一时间触发所有的trigger。这种情况下，你也许想控制你的哪些触发器可以优先获得Quartz的工作线程。为此，你需要设置trigger的优先级属性。如果N个触发器同一时间触发，但是当前仅有Z和可用的工作线程，前Z个具有高优先级的触发器将首先执行。</p>
<p>任何整数，正数或者负数都允许作为优先级。如果你的trigger没有设定优先级它将使用默认的优先级为5。</p>
<p>注意：优先级仅仅在同一时间触发的trigger中才有效。一个在10:59触发的trigger将永远比在11:00触发的trigger先执行。</p>
<p>注意：当触发器被监测到需要恢复时，它的优先级将与原始的优先级相同。</p>
#####触发失败说明
<p>Trigger另一个重要的属性就是“过时触发指令”。如果因为当前scheduler被关闭或者Quartz的线程池没有可用的线程用来执行job导致当前的触发器错过了触发时间，就是出现错误。</p>
<p>不同类型的trigger有不同的失败处理方式。默认情况下，它们使用“智能策略”，其拥有基于trigger类型和配置的动态行为。Scheduler启动时，它将搜索所有失败的持久化的trigger，然后基于它们的失败配置来更新它们。</p>
<p>当你在你的项目中使用Quartz，一定要熟悉它们javaDoc中给定trigger的失败处理策略。更多关于特殊trigger类型的失败策略在第10页“使用trigger”也有提供。</p>
#####日历
<p>Quartz日历对象（不是java.util.Calendar对象）能够与在scheduler中定义和存储的trigger时间相关联。</p>
<p>日历可用于从trigger的触发计划中排除时间块。例如，你可以创建一个trigger在工作日的9:30触发，但是排除法定假日。</p>
<p>日历可以使任何实现了Calendar接口的序列化对象（如下所示）：</p>
```java
package org.quartz;
public interface Calendar {
	public boolean isTimeIncluded(long timeStamp);
	public long getNextIncludedTime(long timeStamp);
}
```
<p>注意这些方法的参数类型是long。这是毫秒的时间戳格式。这意味着日历可以“阻挡”精确至一毫秒的时间。最有可能的是你会对“屏蔽”一整天感兴趣。Quartz包括一个org.quartz.impl.HolidayCalendar就是干这个的，非常方便。</p>
<p>日历必须通过addCalendar(..)方法实例化并注册到scheduler中。如果你使用HolidayCalendar，实例化之后，你应该使用它的addExcludedDate(Date date)方法将你想排除在调度器之外的日期添加进去。同一个日历实例可以被多个trigger使用，如下所示：</p>
```java
		HolidayCalendar cal = new HolidayCalendar();
		cal.addExcludedDate(someDate);
		cal.addExcludedDate(someOtherDate);
		sched.addCalendar("myHolidays", cal, false);
		Trigger t = new Trigger()
				.withIdentity("myTrigger")
				.forJob("myJob")
				.withSchedule(dailyAtHourAndMinute(9, 30)) // execute job daily at 9:30
				.modifiedByCalendar("myHolidays") // but not on holidays
				.build();
		// .. schedule job with trigger
		Trigger t2 = new Trigger()
				.withIdentity("myTrigger2")
				.forJob("myJob2")
				.withSchedule(dailyAtHourAndMinute(11, 30)) // execute job daily at 11:30
				.modifiedByCalendar("myHolidays") // but not on holidays
				.build();
		// .. schedule job with trigger2
```
<p>上面的代码创建了两个触发器，每天都会执行。然而，任何发生在排除日历的触发将被跳过。</p>
<p>org.quartz.impl.calendar包提供了许多Calendar的实现，也许可以满足你的需求。</p>
<h4 id="3.2">SimpleTrigger</h4>
<p>如果你需要一个job在特定的时间执行一次或者在特定的时间，以特定的间隔执行多次SimpleTrigger能够满足你的需求。例如：你想让一个trigger在2015年1月13号11:23:54分触发，或者你想在那个时间执行，执行5次，每10秒一次。</p>
<p>这样描述，你也许不难发现SimpleTrigger的属性：开始时间，结束时间，重复次数，重复间隔。所有这些属性正是你期望的，只有一组与结束时间关联的特殊属性。
重复的次数可以是0，正数或者常数SimpleTrigger.REPEAT_INDEFINITELY。重复的间隔属性必须是0或一个正的长整型，代表毫秒值。注意，间隔为0将导致触发器同时触发“重复次数”这么多次（尽可能接近scheduler的并发管理数）。</p>
<p>如果你还不熟悉Quartz的DateBuilder类，你可能发现依赖于你的开始时间（或结束时间）计算trigger的触发次数很有帮助。</p>
<p>endTIme属性重写了重复次数的属性。如果你想创建一个每10秒触发一次的trigger直到一个给定的时刻，这也许会有用。而不必计算开始时间和结束时间间的重复次数，你可以简单的指出结束时间然后使用REPEAT_INDEFINITELY表明重复次数（你甚至可以指定较大的重复次数超过在结束时间到达之前trigger触发的次数）。SimpleTrigger实例通过使用TriggerBuilder（创建trigger的主属性）和SimpleSchedulerBuilder（创建SimpleTrigger特殊属性）创建。使用静态导入DSL-style使用这些构造工具。</p>
```java
import static org.quartz.TriggerBuilder.*;
import static org.quartz.SimpleScheduleBuilder.*;
import static org.quartz.DateBuilder.*:
```
#####使用不同的Scheduler定义trigger的例子
<p>下面是使用简单的scheduler定义触发器的一些例子。仔细观察，它们每个至少展示了一个新的/不同的点。</p>
<p>同时，花费一些时间查看TriggerBuilder和SimpleSchedulerBuilder定义的所有可用方法以便熟悉这里的示例没展示但是你可能会使用到的选项。</p>
<p>注意：注意如果你不显示的设置属性TriggerBuilder（和其它的quartz创建者）会选取一个合理的值。例如，如果你不调用withIdentity(..)的某个方法，TriggerBuilder将为你的trigger生成一个随机的名字，同样，如果你没有调用startAt(..)，则设定为当前的时间（立即执行）。</p>
<p>创建一个指定时间执行的trigger，没有重复次数</p>
```java
		SimpleTrigger trigger = (SimpleTrigger) new Trigger()
				.withIdentity("trigger1", "group1")
				.startAt(myStartTime) // some Date
				.forJob("job1", "group1") // identify job with name, group strings
			.build();
```
<p>在指定的时间创建一个trigger，然后每10秒触发一次，共触发10次</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger3", "group1")
				.startAt(myTimeToStartFiring) // if a start time is not given (if this line  were omitted), "now" is implied
				.withSchedule(simpleSchedule().withIntervalInSeconds(10).withRepeatCount(10)) // note that 10 repeats will give a total of 11 firings
				.forJob(myJob) // identify job with handle to its JobDetail itself
				.build();
```
<p>创建一个触发一次的trigger，5分钟后触发</p>
```java
		trigger = (SimpleTrigger) new Trigger()
				.withIdentity("trigger5", "group1")
				.startAt(futureDate(5, IntervalUnit.MINUTE)) // use DateBuilder to create a date in the future
				.forJob(myJobKey) // identify job with its JobKey
				.build();
```
<p>创建一个立即触发的trigger，每5分钟触发一次，直到22:00结束</p>
```java
		trigger = new Trigger().withIdentity("trigger7", "group1")
				.withSchedule(simpleSchedule().withIntervalInMinutes(5).repeatForever())
				.endAt(dateOf(22, 0, 0))
			.build();
```
<p>建立一个trigger，每小时开始的时候触发，每2小时出发一次，永远执行</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger8") // because group is not specified, "trigger8" will be in the default group
				.startAt(evenHourDate(null)) // get the next even-hour (minutes and seconds zero ("00:00"))
				.withSchedule(simpleSchedule().withIntervalInHours(2).repeatForever())// note that in this example, 'forJob(..)' is not called - which is valid if the trigger is passed to the scheduler along with the job
				.build();
		scheduler.scheduleJob(trigger, job);
```
#####SimpleTrigger失败说明
<p>SimpleTrigger有几个指令，用来通知quartz当触发失败该怎么做。（关于trigger触发失败的信息，参见trigger章节）这些指令被SimpleTrigger自己定义（它们的行为在Javadoc中有描述）。这些常量包括：</p>

- MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
- MISFIRE_INSTRUCTION_FIRE_NOW
- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT
- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT
- MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT
- MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT

<p>所有的trigger都有MISFIRE_INSTRUCTION_SMART_POLICY属性可以使用，这个指定对所有的触发器类型都是可用的。</p>
<p>如果使用“智能策略”，SimpleTrigger会基于给定SimpleTrigger实例的配置或语句之间动态的选择触发失败指令。SimpleTrigger.updateAfterMisfire()方法的Javadoc解释了这些动态行为的细节。</p>
<p>在创建SimpleTrigger时，你可以指定失败指令作为简单scheduler的一部分（通过SimpleSchedulerBuilder）：</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger7", "group1")
				.withSchedule(simpleSchedule().withIntervalInMinutes(5).repeatForever().withMisfireHandlingInstructionNextWithExistingCount())
			.build();
```
<h4 id="3.3">CronTriggers</h4>
<p>Cron是一个UNIX工具,已经存在了很长时间,所以它的调度功能强大且得到证实。CronTrigger类是基于cron的调度功能。</p>
<p>CronTrigger使用“cron表达式”，其能够像这样创建触发的计划：“每周一至周五早上8:00”或者“每月最后一个周五早上1:30”。</p>
<p>如果你需要基于类似日历的观念反复触发一个job，而不是使用SimpleTrigger指定一个固定的时间间隔，相对于SimpleTrigger，CronTrigger往往更有用。</p>
<p>使用CronTrigger，你可以指定一个触发计划例如“每周五的中午”，或者“每个工作日上午9:00”，甚至“1月的周一，周三和周五早上9:00至10:00每五分钟”。</p>
<p>即便如此，和SimpleTrigger一样， CronTrigger也有一个startTime指定计划的生效时间也有一个（可选的）endTime指定计划的停止时间。</p>
#####Cron表达式
<p>Cron表达式用于CronTrigger实例的配置。Cron表达式实际上是由7个子表达式组成，用来描述计划安排的细节。这些子表达式使用空格分割：</p>

- 秒
- 分
- 时
- 每月的第几天
- 月
- 每周的第几天
- 年（可选字段）

<p>一个完整的cron表达式是一个像这样的字符串“0 0 12 ? WED *”——这代表着“每周三的12:00”。</p>
<p>每个子表达式都能包含范围/或列表，例如前面每周的第几天（“WED”）可以替换成“MON-FRI”，“ MON,WED,FRI”，甚至“MON-WED,SAT”。</p>
<p>可使用通配符（“*”）代表这个字段上的每一个可能的值。因此“月”字段上使用“*”代表着“每个月”。“*”在“每周的第几天”代表“一周的每一天”。</p>
<p>所有的字段都可以指定一个有效值。这些值应该相当明显，例如数字0-59代表秒数和分钟数，0-23代表小时。每月中的第几天可能是1-31，但是你应该注意一个指定的月份应该有多少天！月份可以指定0-11，或者使用字符串JAN, FEB, MAR, APR, MAY, JUN, JUL, AUG, SEP, OCT, NOV和DEC。每周的第几天可以使用1-7（1=周日）或者使用字符串SUN, MON, TUE, WED, THU, FRI和SAT。</p>
<p>“/”符号被用来指定增量值。例如，如果你在分钟字段设置“0/15”，意味着“从0分开始，每15分钟”。如果你在分钟字段使用“3/20”，意味着“从第3分钟开始，每20分钟”——换句话说它和在分钟字段上指定“3,23,43”是一样的。注意“/35”并不是“每35分钟”——它是指“从0开始，每35分钟——换句话说就和“0,35”一样。</p>
<p>“?”允许在每月中的第几天和每周的第几天上设置。它用于指定“无特殊值”。当你需要在这两个字段之一指定一些值，但不能同时设置。具体参见下面示例（和CronTrigger Javadoc）。</p>
<p>“L”允许在每月的第几天和每周的第几天上设置。这个符号是“last”的简写，但是它在这两个字段上的意思是不同的。例如，“L”在每月的第几天字段上指的是“每月的最后一天”——1月31号，和非闰年的2月28号。如果用在每周的第几天，就是“7”或者“SAT”的意思。但是如果用在每周第几天的一个值后面，就意味着“每月最后一个星期的第几天”——例如“6L”或者“FRIL”都表示“每月最后一个周五”。你还可以为每月的最后一天指定一个偏移量，例如“L-3”意味着每月倒数第三天。当使用“L”选项时，重要的是不要指定列表或范围值，不然你将得到混乱/不期望的结果。</p>
<p>“W”用于指定距离给定日期最近的一个工作日（周一至周五）。例如，如果你在每月的第几天字段上指定“15W”，意味着：“离本月15号最近的工作日”。</p>
<p>“#”用于指定每月中的“第几个”周几。例如，在每周的第几天设置“6#3”或者“FRI#3”表示“本月第三个周五”。</p>
#####格式
字段如下：
<table>
	<thead>
		<tr>
			<td>字段名称</td><td>必填项</td><td>允许的值</td><td>允许的特殊字符</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>Seconds</td><td>YES</td><td>0-59</td><td>, - *</td>
		</tr>
		<tr>
			<td>Minutes</td><td>YES</td><td>0-59</td><td>, - *</td>
		</tr>
		<tr>
			<td>Hours</td><td>YES</td><td>0-23</td><td>, - *</td>
		</tr>
		<tr>
			<td>Day of month</td><td>YES</td><td>1-31</td><td>, - * ? / L W</td>
		</tr>
		<tr>
			<td>Month</td><td>YES</td><td>1-12或JAN-DEC</td><td>, - *</td>
		</tr>
		<tr>
			<td>Day of week</td><td>YES</td><td>1-7或SUN-SAT</td><td>, - * ? / L #</td>
		</tr>
		<tr>
			<td>Year</td><td>NO</td><td>空值,1970-2099</td><td>, - *</td>
		</tr>
	</tbody>
</table>

<p>最简单的cron表达式可以写成：* * * * ? *</p>
<p>或者比较复杂的：0/5 14,18,3-39,52 * ? JAN,MAR,SEP MON-FRI 2002-2010</p>
#####特殊字符

- *（所有值）——用来选择一个字段上的所有值。例如：分钟字段上的“*”表示“每分钟”。
- ?（无特定值）——当你需要在允许的两个字段中的一个指定一些值的时候使用。例如，你想让你得trigger在每月的某一天触发（假如，10号），但是不关系在周几发生，你应该在每月的第几天字段上设置“10”，在每周的第几天字段上设置“?”。具体参见下面实例。
- -——用于指定范围。例如，小时字段上设置“10-12”表示“10点，11点，12点”。
- ,——用于指定额外的值。例如，在每周的第几天设置“MON,WED,FRI”表示“周一，周三，周五”。
- /——用于指定增量。例如，秒字段上“0/15”表示“0秒，15秒，30秒，45秒”。秒字段上“5/15”表示“5秒，20秒，35秒，50秒”，你还可以在“character - in this case”后面指定“/”，相当于“/”前有一个“0”，每月中的第几天设置“1/3”表示“从每月的第一天开始每3天触发一次”。
- L（“last”简写）——可用在两个字段每个子段的含义不同。例如，“L”在每月的第几天字段上指的是“每月的最后一天”——1月31号，和非闰年的2月28号。如果用在每周的第几天，就是“7”或者“SAT”的意思。但是如果用在每周第几天的一个值后面，就意味着“每月最后一个星期的第几天”——例如“6L”表示“每月最后一个周五”。你还可以为每月的最后一天指定一个偏移量，例如“L-3”意味着每月倒数第三天。当使用“L”选项时，重要的是不要指定列表或范围值，不然你将得到混乱/不期望的结果。
- W（工作日）——用来指定距离指定日期最近的工作日（周一至周五）。例如，如果你指定了“15W”在每月第几天的字段上，表示：“离本月15号最近的工作日”。所以，如果15号是周六，trigger将在14号周五触发，如果15号是周日，trigger将在16号周一触发，如果15号是周二，trigger将在15号周二触发。然而如果在每月的第几天字段指定“1W”，而且1号是周六，trigger将会在3号周一触发，它不会跳过月的边界。“W”只能指定每月第几天字段中的一天，不能指定一个范围或列表。
- “L”和“W”字符也可以在每月的第几天字段组合成“LW”，表示“每月的最后一个工作日”。
- \#——用于指定每月中的第几个周几。例如，每周第几天字段上的“6#3”表示“每月的第3个周五”（”6“=周五，”#3“=每月的第3个）。另一个例子：”2#1“=每月的第1个周一，”4#5“每月的第5个周三。注意，如果你指定”#5”，但是该月没有第5个这一周的天数，触发器将不会执行。

<p>注意：月份的英文名和周几的英文是不区分大小写的。MON与mon都是合法的字符。</p>
#####示例
<p>这里有一些完整的示例：</p>
<table>
	<thead>
		<tr>
			<td>表达式</td><td>含义</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>0 0 12 * * ?</td><td>每天12:00触发</td>
		</tr>
		<tr>
			<td>0 15 10 ? * *</td><td>每天10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 * * ?</td><td>每天10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 * * ? *</td><td>每天10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 * * ? 2005</td><td>2005年的每天10:15触发</td>
		</tr>
		<tr>
			<td>0 * 14 * * ?</td><td>每天14:00至14:59每分钟触发</td>
		</tr>
		<tr>
			<td>0 0/5 14 * * ?</td><td>每天14:00开始至14:55每5分钟触发</td>
		</tr>
		<tr>
			<td>0 0/5 14,18 * * ?</td><td>每天14:00开始至14:55每5分钟触发，18:00开始至18:55每5分钟触发</td>
		</tr>
		<tr>
			<td>0 0-5 14 * * ?</td><td>每天14:00开始至14:05每分钟触发</td>
		</tr>
		<tr>
			<td>0 10,44 14 ? 3 WED</td><td>三月份的每个周三14:10和14:44触发</td>
		</tr>
		<tr>
			<td>0 15 10 ? * MON-FRI</td><td>每个周一至周五上午10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 15 * ?</td><td>每月15号10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 L * ?</td><td>每月最后一天10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 L-2 * ?</td><td>每月倒数第2天10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 ? * 6L</td><td>每月最后一个周五10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 ? * 6L 2002-2005</td><td>2002年至2005年每月最后一个周五10:15触发</td>
		</tr>
		<tr>
			<td>0 15 10 ? * 6#3</td><td>每月第3个周五10:15触发</td>
		</tr>
		<tr>
			<td>0 0 12 1/5 * ?</td><td>每月的第一天开始，每个第5天12:00触发</td>
		</tr>
		<tr>
			<td>0 11 11 11 11 ?</td><td>每年11月11号11:11触发</td>
		</tr>
	</tbody>
</table>

<p>注意”?”和”*”在每周第几天和每月第几天字段上的影响。</p>
<p>注意</p>

- 支持不需要完全指定一周中的第几天和一月中的第几天（你必须在这两个字段之一使用”?”号）。
- 当设置触发时间在凌晨时注意那些实行夏令时的地区（在美国，时间通常会提前或推后到凌晨2点——因为依赖于时间向前或向后改变会跳过或重复执行触发器。你可以从维基百科查看你的使用环境：https://secure.wikimedia.org/wikipedia/en/wiki/Daylight_saving_time_around_the_world）

#####创建CronTrigger
<p>使用TriggerBuilder（创建trigger的主属性）和CronScheduleBuilder（创建CronTrigger特殊属性）创建CronTrigger实例。用DSL-style使用这些构造类，使用静态导入：</p>
```java
import static org.quartz.TriggerBuilder.*;
import static org.quartz.CronScheduleBuilder.*;
import static org.quartz.DateBuilder.*:
```
<p>创建一个每天8:00至17:00，每分钟执行一次的触发器</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger3", "group1")
				.withSchedule(cronSchedule("0 0/2 8-17 * * ?"))
				.forJob("myJob", "group1").build();
```
<p>创建一个每天10:42触发的触发器</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger3", "group1")
				.withSchedule(dailyAtHourAndMinute(10, 42))
			.forJob(myJobKey)
.build();
```
<p>或</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger3", "group1")
				.withSchedule(cronSchedule("0 42 10 * * ?"))
				.forJob(myJobKey)
			.build();
```
<p>创建一个触发器将在周三10:42触发，基于系统默认的时区</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger3", "group1")
				.withSchedule(weeklyOnDayAndHourAndMinute(DateBuilder.WEDNESDAY, 10, 42))
				.forJob(myJobKey)
				.inTimeZone(TimeZone.getTimeZone("America/Los_Angeles"))
			.build();
```
<p>或</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger3", "group1")
				.withSchedule(cronSchedule("0 42 10 ? * WED"))
				.inTimeZone(TimeZone.getTimeZone("America/Los_Angeles"))
				.forJob(myJobKey)
			.build();
```
#####CronTrigger失败说明
<p>下面的说明可以在CronTrigger失败时用来介绍Quartz信息。（更多关于trigger失败的情况介绍在教程中）。这些介绍是CronTrigger自己定义的常量（包括Javadoc描述的行为）。包括：</p>

- MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
- MISFIRE_INSTRUCTION_DO_NOTHING
- MISFIRE_INSTRUCTION_FIRE_NOW

<p>所有的trigger都有MISFIRE_INSTRUCTION_SMART_POLICY说明可供使用，这个说明也是所有trigger类型默认的。”智能策略CronTrigger解读为“MISFIRE_INSTRUCTION_FIRE_NOW。CronTrigger.updateAfterMisfire()方法的Javadoc解析了这些行为的细节。</p>
<p>在构建CronTrigger时，你可以（通过CronSchedulerBuilder）指定简单schedule的失败说明。</p>
```java
		trigger = new Trigger()
				.withIdentity("trigger3", "group1")
				.withSchedule(cronSchedule("0 0/2 8-17 * * ?")
				.withMisfireHandlingInstructionFireAndProceed())
				.forJob("myJob", "group1")
			.build();
```

<h2 id="4">使用TriggerListener和JobListener</h2>
<h4 id="4.1">TriggerListener和JobListener</h4>
<p>监听器是你基于scheduler中事件的活动创建的对象。就像它们名字所示，TriggerListener接收与trigger有关的事件，JobListener接收与Job有关的事件。</p>
<p>与trigger有关的事件包括trigger触发，trigger触发失败（在第10页“使用trigger”中讨论），trigger执行完成（trigger完成job执行完毕）。</p>
<p>org.quartz.TriggerListener接口</p>
```java
public interface TriggerListener {
	public String getName();

	public void triggerFired(Trigger trigger, JobExecutionContext context);

	public boolean vetoJobExecution(Trigger trigger, JobExecutionContext context);

	public void triggerMisfired(Trigger trigger);

	public void triggerComplete(Trigger trigger, JobExecutionContext context, int triggerInstructionCode);
}
```
<p>与job有关的时间包括：job即将进行的通知，和job已经完成的通知。</p>
<p>org.quartz.JobListener接口</p>
```java
public interface JobListener {
	public String getName();

	public void jobToBeExecuted(JobExecutionContext context);

	public void jobExecutionVetoed(JobExecutionContext context);

	public void jobWasExecuted(JobExecutionContext context, JobExecutionException jobException);
}
```
<h4 id="4.2">创建你自己的监听器</h4>
<p>要创建一个监听器，需要简单创建一个实现org.quartz.TriggerListener接口或/和org.quartz.JobListener接口的对象。</p>
<p>然后在运行时监听器被注册到scheduler中，并且必须给定一个name（或者确切的说，它们必须能够通过它们的getName()方法给出它们的名字）。</p>
<p>为了你的方便，而不需要实现这些接口，你的类还可以继承JobListenerSupport或TriggerListenerSupport并简单的重写你感兴趣的方法。</p>
<p>监听器被scheduler的ListenerManager注册与Matcher一起描述Job/Trigger的监听器想要接受的事件。</p>
<p>注意：监听器在运行时被注册到scheduler，而不是一直随job和trigger存储在JobStore中。这是因为监听器通常作为你的应用程序的一个集成点。因此，你的应用每次运行，监听器都需要重新注册进scheduler。</p>
<p>添加监听器的代码示例</p>
<p>下面的示例演示了将感兴趣的JobListener添加进不同类型的job中。以同样的方式添加可用的TriggerListener。</p>
<p>添加一个感兴趣的JobListener到特定的job</p>
```java
scheduler.getListenerManager().addJobListener(myJobListener,KeyMatcher.jobKeyEquals(new JobKey("myJobName", "myJobGroup")));
```
<p>你可能想使用静态导入导入matcher和key类，这将使你清晰的匹配matcher：</p>
```java
import static org.quartz.JobKey.*;
import static org.quartz.impl.matchers.KeyMatcher.*;
import static org.quartz.impl.matchers.GroupMatcher.*;
import static org.quartz.impl.matchers.AndMatcher.*;
import static org.quartz.impl.matchers.OrMatcher.*;
import static org.quartz.impl.matchers.EverythingMatcher.*;
```
<p>...etc.</p>
<p>上面的例子将变成：</p>
```java
scheduler.getListenerManager().addJobListener(myJobListener,jobKeyEquals(jobKey("myJobName", "myJobGroup")));
```
<p>添加一个感兴趣的JobListener到一个指定group的所有job</p>
```java
scheduler.getListenerManager().addJobListener(myJobListener, jobGroupEquals("myJobGroup"));
```
<p>添加一个感兴趣的JobListener到两个指定的group的所有job</p>
```java
scheduler.getListenerManager().addJobListener(myJobListener, or(jobGroupEquals("myJobGroup"), jobGroupEquals("yourGroup")));
```
<p>添加一个感兴趣的JobListener到所有job</p>
```java
scheduler.getListenerManager().addJobListener(myJobListener, allJobs());
```
<h2 id="5">使用SchedulerListener</h2>
<h4 id="5.1">SchedulerListener</h4>
<p>SchedulerListener和TriggerListener及JobListener很像，除了其接收Scheduler自身事件的通知，而不必与特定的trigger和job相关联。</p>
<p>在大量的其他事件中，Scheduler关联的事件包括：</p>

- 添加一个job或trigger
- 移除一个trigger或job
- Schduler中的一系列错误
- Schduler的关闭

<p>SchedulerListener被scheduler的ListenerManager注册。SchdulerListner几乎可以是任何实现org.quartz.SchedulerListener接口的对象。</p>
<p>org.quartz.SchedulerListener接口</p>
```java
public interface SchedulerListener {
	public void jobScheduled(Trigger trigger);

	public void jobUnscheduled(String triggerName, String triggerGroup);

	public void triggerFinalized(Trigger trigger);

	public void triggersPaused(String triggerName, String triggerGroup);

	public void triggersResumed(String triggerName, String triggerGroup);

	public void jobsPaused(String jobName, String jobGroup);

	public void jobsResumed(String jobName, String jobGroup);

	public void schedulerError(String msg, SchedulerException cause);

	public void schedulerStarted();

	public void schedulerInStandbyMode();

	public void schedulerShutdown();

	public void schedulingDataCleared();
}
```
<h4 id="5.2">添加一个SchedulerListener</h4>
```java
scheduler.getListenerManager().addSchedulerListener(mySchedListener);
```
<h4 id="5.3">移除一个SchedulerListener</h4>
```java
scheduler.getListenerManager().removeSchedulerListener(mySchedListener);
```
<h2 id="6">使用JobStore</h2>
<h4 id="6.1">关于JobStores</h4>
<p>JobStore负责保存你给定的schduler所有的数据：job，trigger，calendar等等。</p>
<p>为你的Quartz scheduler实例在一些重要的步骤选择合适的JobStore。一旦你搞清楚它们之间的不同是很容易的。你声明哪种JobStore根据你的scheduler应该使用属性文件中你提供给你用来生成你的scheduler实例的SchedulerFactory。</p>
<p>你在属性文件(或对象)中声明你的scheduler应该使用哪种JobStore (和其配置设置)，你将其提供给你用来生成scheduler实例的SchedulerFactory。</p>
<p>重点：不要再代码中直接使用你的JobStore实例。JobStore是Quartz自己后台用的。你必须让Quartz（通过配置）知道使用哪种JobStore。之后，你只需在你的代码中使用Scheduler实例。</p>
<h4 id="6.2">RAMJobStore</h4>
<p>RAMJobStore是使用JobStore最简单的方式。它也提供了最好的性能（在CPU时间上）。</p>
<p>RAMJobStore就如它的名字所示，将它的数据保存在RAM中。这也是为什么它快速以及容易配置的原因。</p>
<p>使用RAMJobStore的缺点是，当你的应用程序结束（或者崩溃）后，Scheduler的所有信息都会丢失。因此，RAMJobStore不能担任job和trigger的”持久化“设置。对于某些应用来说，这是可以接受的，甚至是所需要的行为，但是对于另一些应用，可能就是灾难性的。</p>
<p>使用RAMJobStore（假如你使用StdSchedulerFactory）简单的设置org.quartz.jobStore.class属性到org.quartz.simpl.RAMJobStore如下所示：</p>
```java
org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore
```
<p>没有你需要关心的其他设置。</p>
<h4 id="6.3">JDBCJobStore</h4>
<p>JDBCJobStore通过JDBC将所有的数据保存到数据库中，因为它使用数据库，它比RAMJobStore配置更复杂，而且不是那么快。然后，性能的缺点并不是很糟，特别是你建立了数据表上的主键之后。现代的机器配置在相当好的局域网中（在Scheduler和database之间）接收和更新一个trigger通常小于10毫秒。</p>
<p>JDBCJobStore适用于几乎任何数据库,它已经广泛应用于oracle、PostgreSQL、MySQL、MS SQL Serve、HSQLDB和DB2。</p>
<p>使用JDBCJobStore，你必须首先创建一个Quartz使用的数据表。你可以在Quartz的发布包的“docs/dbTables”目录下找到创建表的SQL脚本。如果没有你的数据库的脚本，找一个已经存在的修改成你的数据库需要的形式。</p>
<p>注意，在脚本中所有的表都以前缀” QRTZ_ “开头（例如”QRTZ_TRIGGERS”, 和“QRTZ_JOB_DETAIL”）。这个前缀可以是你想要的任何方式（在你的Quartz属性文件中）。创建不同的前缀可能用于在同一个数据库创建多个scheduler实例，创建多个表。</p>
<p>一旦你创建了表，你需要决定你的应用需要哪种类型的事务。如果你不需要把你的调度命令（如添加和删除trigger）连接到其它的事务上，你可以使用JobStoreTX作为你的JobStore让Quartz管理这些事务（这是最常见的选择）。</p>
<p>如果你需要Quartz和其它事务一起工作（例如，使用J2EE应用服务），你应该使用JobStoreCMT，这种情况下Quartz会让应用服务器管理事务。</p>
</p>最后，你必须设置一个数据源将JDBCJobStore连接到你的数据库。数据源可以在你的Quartz属性文件中使用多种方式之一设置。一种方法就是将数据库的所有连接信息提供给Quartz由Quartz自己创建管理数据源。另一种就是Quartz使用Quartz运行时的应用服务器管理的数据源。你可以通过JNDI的名称提供数据源。关于属性更详细的信息，请参考“docs/config”目录下的配置文件。</p>
<p>使用JDBCJobStore(假设你使用StdSchedulerFactory)首先需要设置Quartz配置的JobStore类属性如下列之一：</p>

- org.quartz.impl.jdbcjobstore.JobStoreTx
- org.quartz.impl.jdbcjobstore.JobStoreCMT

<p>你的选择基于上面一段的解释。</p>
<p>下面的例子展示了如何配置JobStoreTx:</p>
```java
org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
```
<p>接下来，你需要选择一个DriverDelegate供JobStore使用。驱动代理也许需要为你的特定的数据库负责做任何JDBC的工作。</p>
<p>StdJDBCDelegate是一个使用” vanilla”JDBC代码（和SQL语句）来完成工作的代理。如果没有一个特定于你的数据库的代理，就是用这个代理。Quartz为那些使用StdJDBCDelegate不能够操作很好的数据库提供了一些特定的代理。那些代理可以在org.quartz.impl.jdbcjobstore包或其子包下找到。这些代理包括DB2v6Delegate（为了DB2第6版或更早的版本），HSQLDBDelegate（为了HSQLDB），MSSQLDelegate（为了微软SQLServer），PostgreSQLDelegate（为了PostgreSQL），WeblogicDelegate（为了使用Weblogic创建的JDBC驱动），OracleDelegate（为了使用Oracle），等等。</p>
<p>一旦你选择了你的代理，设置它的类名作为代理供JDBCJobStore使用。</p>
<p>下面的实例展示了如何配置JDBCJobStore去使用DriverDelegate:</p>
```java
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
```
<p>接下来，你需要通知JobStore你使用的前缀（如上所示）。</p>
<p>下面的例子展示了如何配置JDBCJobStore的表前缀:</p>
```java
org.quartz.jobStore.tablePrefix = QRTZ_
```
<p>最后，你需要设置JobStore使用哪种数据源。已命名的数据源也要在你得Quartz属性中定义。这种情况下，我们指定Quartz应该使用的数据源名为“myDS”（即在配置属性中定义）。</p>
```java
org.quartz.jobStore.dataSource = myDS
```
<p>提示：如果你的scheduler很繁忙，意味着其总是执行与线程池大小一样的job数，那么你应该将数据源的连接数设置为线程池的大小+2。</p>
<p>提示：org.quartz.jobStore.useProperties的属性可以设置为true（默认为false），通知JDBCJobStore将JobDataMaps中的所有的值转成字符串，因此能够存成键值对的形式，而不是将更复杂的对象以序列化的形式存储在BLOB列中。长期来看这安全多了，因为你避免了从非字符串对象类序列化到BLOB的类版本问题。</p>
<h4 id="6.4">TerracottaJobStore</h4>
<p>TerracottaJobStore为无需使用数据库进行扩展和鲁棒性提供了手段。这意味着你的数据库可以与Quartz保持自由的加载，反而可以将它的所有资源保存为你的应用的一部分。</p>
<p>TerracottaJobStore能够在集群或者非集群上运行，这两种情况下为你的job数据持久化在应用重启时提供了存储介质，因为数据存储在Terracotta server上。它的性能比通过JDBCJobStore使用一个数据库要慢（大概好一个数量级），但是比RAMJobStore要慢。</p>
<p>在你的Quartz配置中简单的设置org.quartz.jobStore.class属性为org.terracotta.quartz.TerracottaJobStore就可以使用TerracottaJobStore，还需要添加一个额外的属性到Terracotta server上：</p>
```java
org.quartz.jobStore.class = org.terracotta.quartz.TerracottaJobStore
org.quartz.jobStore.tcConfigUrl = localhost:9510
```
<p>关于TerracottaJobStore的更多信息，参见Terracotta Quartz用户手册。</p>
 
<h2 id="7">配置，Scheduler Factory和日志</h2>
<h4 id="7.1">配置组件</h4>
<p>Quartz的体系结构是模块化的，因此想运行它需要把几个组建“结合”在一起。幸运的是，有很多已存在的助手完成这件事。</p>
<p>在Quartz能工作之前的主要需要配置的组件是：</p>

- ThreadPool
- JobStore
- 数据源 (如果必须的话)
- Scheduler

<p>ThreadPool为Quartz执行job提供了一组线程。线程池中的线程越多，可以同时执行的job就越多。然而，太多的线程将会拖慢你的系统。大多数Quartz使用者发现大约5个线程就足够了。因为他们做任何时间给定的job数都少于100，job通常不会同时执行，而且job运行时间很短（完成很快）。</p>
<p>其他用户发现他们需要15，50或者100个线程,因为他们各种各样的schduler有成千上万的trigger,最终在任何时刻平均有10到100个job在执行。</p>
<p>找到正确的scheduler池的大小完全依赖于你使用scheduler做什么。没有真正的规则可循，除了尽可能小的保持线程数（参考你的机器资源）。不过，你要确保你要确保你的job能够准时触发。注意，如果一个trigger时间到了，但是没有足够的可用线程，Quartz将会冻结（暂停）直到有一个可用的线程，那么job将会比它应该执行的时间晚一些。如果在scheduler配置了“失败阀值”期间没有可用线程，这甚至会导致线程失败。</p>
<p>ThreadPool接口的定义在org.quartz.spi包中，你可以使用你喜欢的任何方式创建ThreadPool。Quartz附带了一个简单的ThreadPool叫org.quartz.simpl.SimpleThreadPool。这个org.quartz.simpl.SimpleThreadPool.简单的维护一组固定的线程池，不会增加也不会减少。但是它十分健壮和有很好的测试。几乎每个使用Quartz的用户都是用它。</p>
<p>JobStores和DataSources的讨论在第23页“使用JobStores”。值得注意的是，所有的JobStores都实现了org.quartz.spi.JobStore接口，如果内置的JobStores不符合你得需求，你可以创建自己的。</p>
<p>最后，你需要创建你自己的scheduler实例。Scheduler实例需要给定一个名字，通知RMI设置，传入一个JobStore的实例和ThreadPool。RMI设置包括scheduler是否应该被创建成一个RMI的服务器对象（是其可用于远程连接），使用什么样的地址和端口等等。StdSchedulerFactory（下面讨论）也可以远程创建一个scheduler实例，实际上是一个代理（RMI stubs）。更多关于StdSchedulerFactory的信息，参见第26页的"Scheduler工厂"</p>
<h4 id="7.2">Scheduler工厂</h4>
<p>StdSchedulerFactory是org.quartz.SchedulerFactory接口的一个实现。它使用一组属性（java.util.Properties）来创建Quartz Scheduler实例。这些属性通常从一个文件存储和加载，也可以由你的程序直接传给工厂。简单的调用工厂的getScheduler()将产生一个Scheduler，对其初始化（ThreadPool, JobStore和数据源）并返回它的公共接口的句柄。</p>
<p>有一些简单的配置示例（包括属性说明）在Quartz发布包的“docs/config”目录下。你可以在Quartz的示例程序和示例代码中找到完整的文档。</p>
#####DirectSchedulerFactory
<p>DirectSchedulerFactory是另一个SchedulerFactory实现。如果你想用更程序化的方法创建你自己的Scheduler实例，它会有帮助。基于下面两个原因一般不鼓励使用它：（1）它需要深入理解Scheduler（2）它不允许使用配置（意味着你必须硬编码Scheduler的配置）。</p>
<h4 id="7.3">日志</h4>
<p>Quartz使用SLF4J框架满足日志需求。为了“优化”框架设置（例如输出数量，输出的地方），你需要理解SLF4J框架，这超出了本文范围。</p>
<p>如果你想获得trigger触发和job执行的详细信息，你可能对org.quartz.plugins.history.LoggingJobHistoryPlugin 和/或org.quartz.plugins.history.LoggingTriggerHistoryPlugin感兴趣。</p>
 
<h2 id="8">Quartz调度器的其他特性</h2>
<h4 id="8.1">插件</h4>
<p>Quartz提供org.quartz.spi.SchedulerPlugin接口为插件提供附加功能。</p>
<p>在org.quartz.plugins包中提供了Quartz附带插件各种实用功能的描述。</p>
<p>插件提供的功能，如scheduler启动后自动安排计划，记录job和trigger事件的历史记录，确保在JVM关闭时scheduler关闭干净。</p>
<h4 id="8.2">JobFactory</h4>
<p>当触发器触发时，在scheduler中相关的job通过JobFactory配置被实例化。Job类上的JobFactory默认调用newInstance()。你也许想创建你自己的JobFactory实例完成一些事诸如在你的IoC或DI容器中产生/初始化job实例。</p>
观察org.quartz.spi.JobFactory接口，和相关的Scheduler.setJobFactory(fact)方法。
<h4 id="8.3">Factory自带Job</h4>
<p>Quartz提供了大量实例的job，你可以在你的应用中实现诸如发送短信执行EJB之类的事。</p>
<p>你可以在org.quartz.jobs包中找到它们的文档</p>
<h2 id="9">高级功能</h2>
<h4 id="9.1">集群</h4>
<p>集群当前可以使用JDBC-Jobstore(JobStoreTX或JobStoreCMT)和TerracottaJobStore工作。包括负载平衡和故障转移功能（如果JobDetail的“请求复苏”标志设置为true）。</p>
使用JobStoreTX或JobStoreCMT集群
<p>通过设置org.quartz.jobStore.isClustered为true来使用集群。集群中的每个实例应该使用相同的quartz.properties文件，以下情况可以例外：线程池的大小不同，org.quartz.scheduler.instanceId的属性不同。集群中的每个节点必须有一个唯一的instanceId，通过设置“AUTO”作为属性值这很容易实现（不需要不同的属性文件）。</p>
<p>重点：不要在单独的机器上运行集群，除非它们非常定期的使用某种时间同步服务或守护进程同步它们的时钟（每个时钟相差在一秒内）。如果你不清楚怎么做，参见http://www.boulder.nist.gov/timefreq/service/its.htm。</p>
<p>重点：不要在非集群实例上将多个运行的实例设置为相同的表设置。你可能得到损坏的数据，并得到不可预测的行为。</p>
<p>每次执行只有一个节点触发job。例如，如果job重复触发通知它每10秒触发一次，12:00:00有个节点会运行job，12:00:10也会有一个节点将运行job。每次运行的节点不一定相同——会或多或少的随机选择运行的节点。负载均衡机制就是为繁忙的scheduler（很多trigger）接近随机调度，仅仅在scheduler不繁忙的时候（一两个trigger）容易使用同一个节点。</p>
#####TerracottaJobStore集群
<p>使用TerracottaJobStore简单的配置scheduler的描述在第25页“TerracottaJobStore”中，你得scheduler将被设置为集群。</p>
<p>你也要考虑你设置Terracotta server的影响，特别是那些功能选项，例如持久化，为HA运行的Terracotta server阵列。</p>
<p>TerracottaJobStore授权版提供了Quartz的高级特性，允许智能的为job目标提供合适的节点。</p>
<h4 id="9.2">JTA事务</h4>
<p>JobStoreCMT允许Quartz调度操作更大的JTA事务执行。</p>
<p>通过设置org.quartz.scheduler.wrapJobExecutionInUserTransaction为true，job也可以在JTA事务（用户事务）中执行。设置了这个选项，JTA事务将在job执行方法调用前执行begin()，在执行完成之后执行commit()。这适用于所有的job。</p>
<p>如果你想标明每个job是否应该在JTA事务中执行，你应该在job类上使用@ExecuteInJTATransaction注解。</p>