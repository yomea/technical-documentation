## 一、回顾

```
public class Test {

    public static void main(String[] args) {

        Scheduler scheduler = new Scheduler();
        scheduler.schedule("*/1 * * * *", new Task() {
            @Override
            public void execute(TaskExecutionContext context) throws RuntimeException {
                System.out.println("hello world!");
            }
        });
        scheduler.start();

    }

}
```

上面这段代码表示每分钟向控制台打印一次hello world

## 二、start

```
public void it.sauronsoftware.cron4j.Scheduler#start() throws IllegalStateException {
    //锁定，避免多次调用start方法
	synchronized (lock) {
	    //多次启动start方法将抛出错误
		if (started) {
			throw new IllegalStateException("Scheduler already started");
		}
		// Initializes required lists.
		//这个集合用于记录正在循环匹配任务的任务巡逻线程
		launchers = new ArrayList();
		//当一个任务匹配当前调度时间时，将创建一个执行器，用于执行任务
		executors = new ArrayList();
		// Starts the timer thread.
		//启动每分钟轮询线程
		timer = new TimerThread(this);
		timer.setDaemon(daemon);
		timer.start();
		// Change the state of the scheduler.
		started = true;
	}
}
```

## 三、TimerThread

TimerThread继承了Thread，它是一个线程类

### 3.1 run

```
public void run() {
	// What time is it?
	long millis = System.currentTimeMillis();
	// Calculating next minute.
	//计算一分钟后的时间
	long nextMinute = ((millis / 60000) + 1) * 60000;
	// Work until the scheduler is started.
	for (;;) {
		// Coffee break 'till next minute comes!
		long sleepTime = (nextMinute - System.currentTimeMillis());
		//如果还没有到下一分钟的时间，那么继续睡
		if (sleepTime > 0) {
			try {
			    //这个方法尽量保证睡足sleepTime，在sleep时，如果被提前唤醒，那么会继续sleep
				safeSleep(sleepTime);
			} catch (InterruptedException e) {
				// Must exit!
				break;
			}
		}
		// What time is it?
		millis = System.currentTimeMillis();
		// Launching the launching thread!
		//启动匹配cron表达式的线程去巡查是否存在匹配当前时间的任务
		scheduler.spawnLauncher(millis);
		// Calculating next minute.
		//重新计算下一分钟的时间
		nextMinute = ((millis / 60000) + 1) * 60000;
	}
	// Discard scheduler reference.
	scheduler = null;
}
```

从上面的一段代码可知，cron4j的轮询线程是每隔一分钟执行一次的，为什么是没分钟执行一次呢？因为在cron4j中，最小的调度时间就是1分钟，如果我们需要增加一个秒呢？
我们可以扩展SchedulingPattern，增加秒时间字段的解析，然后将当前轮询线程变成每秒轮询一次即可。

### 3.2 safeSleep


```
private void safeSleep(long millis) throws InterruptedException {
	long done = 0;
	do {
		long before = System.currentTimeMillis();
		//一开始睡millis毫秒
		sleep(millis - done);
		long after = System.currentTimeMillis();
		//如果因为系统时钟原因，比如系统时间被别人提前了，那么会提前醒来
		//此时可以检查是否已经睡够一分钟，如果没有继续睡
		done += (after - before);
	} while (done < millis);
}
```

## 四、spawnLauncher

```
LauncherThread spawnLauncher(long referenceTimeInMillis) {
	TaskCollector[] nowCollectors;
	synchronized (collectors) {
		int size = collectors.size();
		//用于记录当前收集器中存在的任务
		//收集器中的任务是可以随时添加的，所以这里锁定collectors只获取当前size的任务
		nowCollectors = new TaskCollector[size];
		for (int i = 0; i < size; i++) {
			nowCollectors[i] = (TaskCollector) collectors.get(i);
		}
	}
	//启动LauncherThread，用于巡查收集器中的任务是否匹配当前时间
	LauncherThread l = new LauncherThread(this, nowCollectors,
			referenceTimeInMillis);
	synchronized (launchers) {
	    //记录当前正在巡查匹配任务的线程对象
		launchers.add(l);
	}
	l.setDaemon(daemon);
	l.start();
	return l;
}
```
启动LauncherThread线程，下面我们来看看它的run方法

```
public void LauncherThread#run() {
	outer: for (int i = 0; i < collectors.length; i++) {
	    //获取每个收集器中的所有任务
		TaskTable taskTable = collectors[i].getTasks();
		int size = taskTable.size();
		for (int j = 0; j < size; j++) {
		    //如果当前线程已经被中断，那么退出
			if (isInterrupted()) {
				break outer;
			}
			//获取当前任务的调度模式进行匹配
			SchedulingPattern pattern = taskTable.getSchedulingPattern(j);
			if (pattern.match(scheduler.getTimeZone(), referenceTimeInMillis)) {
			    //如果匹配，去除任务，启动一个新的线程去执行它
				Task task = taskTable.getTask(j);
				scheduler.spawnExecutor(task);
			}
		}
	}
	// Notifies completed.
	//用于移除记录的Launcher线程
	scheduler.notifyLauncherCompleted(this);
}
```

launch线程循环每个收集器中的任务，用当前传入的时间去匹配对应任务的cron表达式，如果匹配那么将取出任务进行执行。

```
public boolean it.sauronsoftware.cron4j.SchedulingPattern#match(java.util.TimeZone, long) {
    GregorianCalendar gc = new GregorianCalendar();
	gc.setTimeInMillis(millis);
	gc.setTimeZone(timezone);
	int minute = gc.get(Calendar.MINUTE);
	int hour = gc.get(Calendar.HOUR_OF_DAY);
	int dayOfMonth = gc.get(Calendar.DAY_OF_MONTH);
	int month = gc.get(Calendar.MONTH) + 1;
	int dayOfWeek = gc.get(Calendar.DAY_OF_WEEK) - 1;
	int year = gc.get(Calendar.YEAR);
	for (int i = 0; i < matcherSize; i++) {
	    //获取每个时间段的匹配器进行匹配，只有所有时间字段都匹配才返回true
		ValueMatcher minuteMatcher = (ValueMatcher) minuteMatchers.get(i);
		ValueMatcher hourMatcher = (ValueMatcher) hourMatchers.get(i);
		ValueMatcher dayOfMonthMatcher = (ValueMatcher) dayOfMonthMatchers.get(i);
		ValueMatcher monthMatcher = (ValueMatcher) monthMatchers.get(i);
		ValueMatcher dayOfWeekMatcher = (ValueMatcher) dayOfWeekMatchers.get(i);
		boolean eval = minuteMatcher.match(minute)
				&& hourMatcher.match(hour)
				&& ((dayOfMonthMatcher instanceof DayOfMonthValueMatcher) ? ((DayOfMonthValueMatcher) dayOfMonthMatcher)
						.match(dayOfMonth, month, gc.isLeapYear(year))
						: dayOfMonthMatcher.match(dayOfMonth))
				&& monthMatcher.match(month)
				&& dayOfWeekMatcher.match(dayOfWeek);
		if (eval) {
			return true;
		}
	}
	return false;
}
```

## 五、spawnExecutor

```
TaskExecutor it.sauronsoftware.cron4j.Scheduler#spawnExecutor(Task task) {
    //创建任务执行器
	TaskExecutor e = new TaskExecutor(this, task);
	synchronized (executors) {
	    //记录任务执行器
		executors.add(e);
	}
	e.start(daemon);
	return e;
}

void it.sauronsoftware.cron4j.TaskExecutor#start(boolean daemon) {
	synchronized (lock) {
		startTime = System.currentTimeMillis();
		String name = "cron4j::scheduler[" + scheduler.getGuid() + "]::executor[" + guid + "]";
		thread = new Thread(new Runner());
		thread.setDaemon(daemon);
		thread.setName(name);
		thread.start();
	}
}


private class Runner implements Runnable {

	/**
	 * It implements {@link Thread#run()}, executing the wrapped task.
	 */
	public void run() {
		Throwable error = null;
		startTime = System.currentTimeMillis();
		try {
			// Notify.
			//任务执行前调用监听器
			scheduler.notifyTaskLaunching(myself);
			// Task execution.
			//任务执行
			task.execute(context);
			// Succeeded.
			//任务调用成功后调用监听器
			scheduler.notifyTaskSucceeded(myself);
		} catch (Throwable exception) {
			// Failed.
			error = exception;
			//任务调用失败监听器
			scheduler.notifyTaskFailed(myself, exception);
		} finally {
			// Notify.
			//任务停止监听器
			notifyExecutionTerminated(error);
			//从Scheduler对象中移除当前执行器
			scheduler.notifyExecutorCompleted(myself);
		}
	}
}
```
