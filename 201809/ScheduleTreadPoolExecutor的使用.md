 ScheduledThreadPool（4个里面唯一一个有延迟执行和周期重复执行的线程池）

```

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize){  
     return new ScheduledThreadPoolExecutor(corePoolSize);  
}  
public ScheduledThreadPoolExecutor(int corePoolSize){  
 super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedQueue ());  
 }  
 //使用，延迟1秒执行，每隔2秒执行一次Runnable r  
Executors. newScheduledThreadPool (5).scheduleAtFixedRate(r, 1000, 2000, TimeUnit.MILLISECONDS);
```  

（1）核心线程数固定，非核心线程（闲着没活干会被立即回收）数没有限制。
（2）从上面代码也可以看出，ScheduledThreadPool主要用于执行定时任务以及有固定周期的重复任务。
 (3) 使用并队列-无界阻塞延迟队列DelayQueue
问题1：为什么不使用Timer和TimeTask

1.因为Timer有误差，大概延时1-2毫秒。
2.Timer只创建了一个线程。当你认为执行的时间超过设置的延时时间将会产生一些不可控制的问题
3.Timer创建的线程没有处理异常，一旦抛出非受检的一次，该线程会立即停止，而ScheduleQueue在出现currentSize>coorSize，且workQueue队列满的时候，会抛出RejectException。



```
private static final String TAG = "ScheduleQueue";
    private ScheduledThreadPoolExecutor exec = null;

    private static class SingletonHolder {
        private static final ScheduleQueue instance = new ScheduleQueue();
    }

    private ScheduleQueue() {
        exec = new ScheduledThreadPoolExecutor(5);
    }

    public static ScheduleQueue getInstance() {
        return SingletonHolder.instance;
    }

    /**
     * 单次执行任务.
     * 
     * @param command Runnable.
     * @return 任务添加状态 true/false.
     */
    public boolean addSchedule(Runnable command) {

        try {
            exec.execute(command);
            return true;
        } catch (Throwable t) {
            // GBDLog.log(TAG, t.toString());
        }
        return false;
    }

    /**
     * 单次延时执行任务.
     * 
     * @param command Runnable.
     * @param delay 延时执行时间.
     * @return 任务添加状态 true/false.
     */
    public boolean addSchedule(Runnable command, long delay) {

        try {
            exec.schedule(command, delay, TimeUnit.MILLISECONDS);
            return true;
        } catch (Throwable t) {
            // GBDLog.log(TAG, t.toString());

        }
        return false;
    }

    /**
     * 循环执行任务.
     * 
     * @param command Runnable.
     * @param initialDelay 初始化延时执行时间.
     * @param period 循环执行时间间隔.
     * @return 任务添加状态 true/false.
     */
    public boolean addSchedule(Runnable command, long initialDelay, long period) {

        try {
            exec.scheduleAtFixedRate(command, initialDelay, period, TimeUnit.MILLISECONDS);
            return true;
        } catch (Throwable t) {
            // GBDLog.log(TAG, t.toString());

        }
        return false;
    }

    /**
     * 循环执行任务.
     *
     * @param command Runnable.
     * @param initialDelay 初始化延时执行时间.
     * @param period 循环执行时间间隔.
     * @return ScheduledFuture.
     */
    public ScheduledFuture addScheduler(Runnable command, long initialDelay, long period) {
        try {
            return exec.scheduleAtFixedRate(command, initialDelay, period, TimeUnit.MILLISECONDS);
        } catch (Throwable t) {
            // GBDLog.log(TAG, t.toString());
        }
        return null;
    }
}
```

原理：
 1.把传入的 Runnable/Callback 装饰为 delay 队列需要的格式的元素
 2.然后把元素加入到阻塞队列
 3.线程池会从阻塞队列获取超时的元素任务进行处理
 


