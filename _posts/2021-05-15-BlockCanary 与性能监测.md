---
layout:     post  
title:      "BlockCanary 与性能监测"  
subtitle:   "帮你监控性能"  
date:       2021-05-15 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - 性能优化 
    - 三方库 

---


## 参考

Github：https://github.com/markzhai/AndroidPerformanceMonitor

据此 Github 主说，起这个名是为了致敬 LeakCanary；

## 使用


```
public class DemoApplication extends Application {
    @Override
    public void onCreate() {
        // ...
        // Do it on main process
        BlockCanary.install(this, new AppBlockCanaryContext()).start();
    }
}
```

install 函数需要一个 AppBlockCanaryContext 类的对象来获取配置信息。
AppBlockCanaryContext 实现如下：

```
public class AppBlockCanaryContext extends BlockCanaryContext {

    /**
     * Implement in your project.
     *
     * @return Qualifier which can specify this installation, like version + flavor.
     */
    public String provideQualifier() {
        return "unknown";
    }

    /**
     * Config block threshold (in millis), dispatch over this duration is regarded as a BLOCK. You may set it
     * from performance of device.
     *
     * @return threshold in mills
     */
    public int provideBlockThreshold() {
        return 1000;
    }

    。。。

    /**
     * Block interceptor, developer may provide their own actions.
     */
    public void onBlock(Context context, BlockInfo blockInfo) {

    }
}
```

## 工作原理

![](/img/media/16214397028030/16214412682470.jpg)

文字版的原理描述：
给 Looper 设置一个 printer；在主线程的消息循环中，在分发每一个消息的前后，都会调用 
printer 的 print 函数，打印一串字符；
我们可以拓展 Printer 类，重写 print 函数，加上自己的代码，就可以统计分发消息的前后的时间，根据时间差是否超过阈值来判断是否卡顿；
如果超过阈值，就记录堆栈，debug 发送通知，release 上报后台。

## 源码探究

从该行代码开始跟踪：

```
    BlockCanary.install(this, new AppBlockCanaryContext()).start();
```

start 函数：

```
// com/github/moduth/blockcanary/BlockCanary.java

    /**
     * Start monitoring.
     */
    public void start() {
        if (!mMonitorStarted) {
            mMonitorStarted = true;
            Looper.getMainLooper().setMessageLogging(mBlockCanaryCore.monitor);
        }
    }
```
### setMessageLogging

```
// Looper.java

    private Printer mLogging;
    
    public void setMessageLogging(@Nullable Printer printer) {
        mLogging = printer;
    }
```

设置一个 printer 给 mLogging 字段。
然后在 Looper.java 的 loop 函数中的消息循环处理中：

```
// Looper.java

    // msg 处理前
    final Printer logging = me.mLogging;
    if (logging != null) {
                    logging.println(">>>>> Dispatching to " + msg.target + " " +
                            msg.callback + ": " + msg.what);
                }
                                            
    // msg 处理中
    msg.target.dispatchMessage(msg);
    
    // msg 处理后
    if (logging != null) {
                    logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
                }
```

### LooperMonitor#println

再看看设置进去的 mBlockCanaryCore.monitor，如何重写 println 函数？

```
// LooperMonitor.java

    @Override
    public void println(String x) {
        if (mStopWhenDebugging && Debug.isDebuggerConnected()) {
            return;
        }
        if (!mPrintingStarted) {
            // 1，记录开始时间
            mStartTimestamp = System.currentTimeMillis();
            mStartThreadTimestamp = SystemClock.currentThreadTimeMillis();
            mPrintingStarted = true;
            // 2，开始记录堆栈
            startDump();
        } else {
            // 3，记录结束时间 
            final long endTime = System.currentTimeMillis();
            mPrintingStarted = false;
            // 4，判断是否卡顿，调用 notifyBlockEvent 函数处理
            if (isBlock(endTime)) {
                notifyBlockEvent(endTime);
            }
            // 5，结束记录堆栈
            stopDump();
        }
    }

    // 6，判断卡顿
    private boolean isBlock(long endTime) {
        return endTime - mStartTimestamp > mBlockThresholdMillis;
    }
    
    // 7，发起通知
    private void notifyBlockEvent(final long endTime) {
        final long startTime = mStartTimestamp;
        final long startThreadTime = mStartThreadTimestamp;
        final long endThreadTime = SystemClock.currentThreadTimeMillis();
        HandlerThreadFactory.getWriteLogThreadHandler().post(new Runnable() {
            @Override
            public void run() {
                mBlockListener.onBlockEvent(startTime, endTime, startThreadTime, endThreadTime);
            }
        });
    }
    
    // onBlockEvent 会调用到类的 onBlock 方法
    // 第一，BlockCanaryContext
    // 第二，AppBlockCanaryContext，即 BlockCanaryContext 的拓展类
    // 上述初始化函数 BlockCanary.install() 中就是传入 AppBlockCanaryContext 对象
    // 第三，DisplayService，显示服务类
```

在注释1、2、3、4、5处，是我关注的主要逻辑：记录开始结束时间，并记录这一段的调用堆栈；
在注释4处，调用 isBlock 函数判断卡顿；
如果卡顿，进一步调用注释7 的 notifyBlockEvent 函数发起通知，最终会调用到
DisplayService 的 onBlock 方法；

### 判断卡顿

追踪上述注释4的源码：

```
// LooperMonitor.java
    
    private boolean isBlock(long endTime) {
        return endTime - mStartTimestamp > mBlockThresholdMillis;
    }
    
    private static final int DEFAULT_BLOCK_THRESHOLD_MILLIS = 3000;

    private long mBlockThresholdMillis = DEFAULT_BLOCK_THRESHOLD_MILLIS;
    
    // 构造函数中设置过 mBlockThresholdMillis
    public LooperMonitor(BlockListener blockListener, long blockThresholdMillis, boolean stopWhenDebugging) {
        if (blockListener == null) {
            throw new IllegalArgumentException("blockListener should not be null.");
        }
        mBlockListener = blockListener;
        // 注意这里哟，不是默认 3 秒哟！
        mBlockThresholdMillis = blockThresholdMillis;
        mStopWhenDebugging = stopWhenDebugging;
    }
    
    // 调用 LooperMonitor 的构造函数的地方
    // BlockCanaryInternals.java
    public BlockCanaryInternals() {

        ...

        setMonitor(new LooperMonitor(new LooperMonitor.BlockListener() {

            @Override
            public void onBlockEvent(long realTimeStart, long realTimeEnd,
                                     long threadTimeStart, long threadTimeEnd) {
                // Get recent thread-stack entries and cpu usage
                ArrayList<String> threadStackEntries = stackSampler
                        .getThreadStackEntries(realTimeStart, realTimeEnd);
                if (!threadStackEntries.isEmpty()) {
                    BlockInfo blockInfo = BlockInfo.newInstance()
                            .setMainThreadTimeCost(realTimeStart, realTimeEnd, threadTimeStart, threadTimeEnd)
                            .setCpuBusyFlag(cpuSampler.isCpuBusy(realTimeStart, realTimeEnd))
                            .setRecentCpuRate(cpuSampler.getCpuRateInfo())
                            .setThreadStackEntries(threadStackEntries)
                            .flushString();
                    LogWriter.save(blockInfo.toString());

                    if (mInterceptorChain.size() != 0) {
                        for (BlockInterceptor interceptor : mInterceptorChain) {
                            interceptor.onBlock(getContext().provideContext(), blockInfo);
                        }
                    }
                }
            }
        }, 
        getContext().provideBlockThreshold(), // 这里传入了卡顿的判断阈值
        getContext().stopWhenDebugging()));
        ...
    }
    
    // BlockCanaryContext.java
    public int provideBlockThreshold() {
        return 1000; // 可知，卡顿的判断阈值时 1000ms
    }
```

判断卡顿的逻辑非常简单，只要超过 1 秒，即认为是卡顿。

### 记录堆栈

```
// LooperMonitor.java

    private void startDump() {
        if (null != BlockCanaryInternals.getInstance().stackSampler) {
            // 1
            BlockCanaryInternals.getInstance().stackSampler.start();
        }

        if (null != BlockCanaryInternals.getInstance().cpuSampler) {
            // 2
            BlockCanaryInternals.getInstance().cpuSampler.start();
        }
    }

    private void stopDump() {
        if (null != BlockCanaryInternals.getInstance().stackSampler) {
            // 3
            BlockCanaryInternals.getInstance().stackSampler.stop();
        }

        if (null != BlockCanaryInternals.getInstance().cpuSampler) {
            // 4
            BlockCanaryInternals.getInstance().cpuSampler.stop();
        }
    }
    
    // 这两个类，都有一个父类 AbstractSampler
    BlockCanaryInternals.getInstance().stackSampler
    BlockCanaryInternals.getInstance().cpuSampler
    
    // AbstractSampler.java
    
    public void start() {
        if (mShouldSample.get()) {
            return;
        }
        mShouldSample.set(true);

        HandlerThreadFactory.getTimerThreadHandler().removeCallbacks(mRunnable);
        HandlerThreadFactory.getTimerThreadHandler().postDelayed(mRunnable,
                BlockCanaryInternals.getInstance().getSampleDelay());
    }
    
    // 延迟多少秒？
    long getSampleDelay() {
        // 其实返回的就是 1000 * 0.8f = 800 ms
        return (long) (BlockCanaryInternals.getContext().provideBlockThreshold() * 0.8f);
    }
```

注释1，2的两个 start 函数，最终会调用到 StackSampler.java 和 CpuSampler.java 的 doSample 函数，开始记录信息；doSample 函数是在 mRunnable 中调用的，也就是在某个子线程中执行的。

注释3，4的两个 stop 函数，就是把执行上述 doSample 函数的 mRunnable 移除。

那么 mRunnable 再哪个线程执行？

```
// AbstractSampler.java

    // 这个 handler 来 post 和 removeCallbacks
    HandlerThreadFactory.getTimerThreadHandler()
```

```
// HandlerThreadFactory.java

    private static HandlerThreadWrapper sLoopThread = new HandlerThreadWrapper("loop");
    private static HandlerThreadWrapper sWriteLogThread = new HandlerThreadWrapper("writer");

    public static Handler getTimerThreadHandler() {
        return sLoopThread.getHandler();
    }

    public static Handler getWriteLogThreadHandler() {
        return sWriteLogThread.getHandler();
    }
    
    // 一个关联子线程的 handler
    private static class HandlerThreadWrapper {
        private Handler handler = null;

        public HandlerThreadWrapper(String threadName) {
            HandlerThread handlerThread = new HandlerThread("BlockCanary-" + threadName);
            handlerThread.start();
            handler = new Handler(handlerThread.getLooper());
        }

        public Handler getHandler() {
            return handler;
        }
    }
```

#### 具体实现

```
// StackSampler.java

    @Override
    protected void doSample() {
        StringBuilder stringBuilder = new StringBuilder();
        
        // 获取当前线程的堆栈，记录到 StringBuilder
        for (StackTraceElement stackTraceElement : mCurrentThread.getStackTrace()) {
            stringBuilder
                    .append(stackTraceElement.toString())
                    .append(BlockInfo.SEPARATOR);
        }

        synchronized (sStackMap) {
            // 维持一个最大存储量
            if (sStackMap.size() == mMaxEntryCount && mMaxEntryCount > 0) {
                // 从最早那个移除
                sStackMap.remove(sStackMap.keySet().iterator().next());
            }
            // 每一次的堆栈，都记录到 sStackMap 中
            sStackMap.put(System.currentTimeMillis(), stringBuilder.toString());
        }
    }

// CpuSampler.java

    @Override
    protected void doSample() {
        BufferedReader cpuReader = null;
        BufferedReader pidReader = null;

        try {
            // 从 “/proc/stat” 文件读取 cpu 数值 
            cpuReader = new BufferedReader(new InputStreamReader(
                    new FileInputStream("/proc/stat")), BUFFER_SIZE);
            // cpu 的总占用率
            String cpuRate = cpuReader.readLine();
            if (cpuRate == null) {
                cpuRate = "";
            }

            if (mPid == 0) {
                mPid = android.os.Process.myPid();
            }
            // 从 “/proc/mPid/stat” 文件读取当前进程的 cpu 占用率 
            pidReader = new BufferedReader(new InputStreamReader(
                    new FileInputStream("/proc/" + mPid + "/stat")), BUFFER_SIZE);
            // 当前进程的 cpu 占用率
            String pidCpuRate = pidReader.readLine();
            if (pidCpuRate == null) {
                pidCpuRate = "";
            }
            
            // 近一步解析这两个值 cpuRate, pidCpuRate
            parse(cpuRate, pidCpuRate);
            // 在函数内，把解析的结果用 StringBuilder 拼接，
            // 并存到 mCpuInfoEntries 变量
        } catch (Throwable throwable) {
            Log.e(TAG, "doSample: ", throwable);
        } finally {
            ..
        }
    }
```

小结：
- 从 `mCurrentThread.getStackTrace()` 拿到堆栈；
- 从 “/proc/stat” 文件读取「cpu 总占用率」；
- 从 “/proc/mPid/stat” 文件读取「当前进程的 cpu 占用率」。

### 发通知

```
// DisplayService.java

    @Override
    public void onBlock(Context context, BlockInfo blockInfo) {
        Intent intent = new Intent(context, DisplayActivity.class);
        intent.putExtra("show_latest", blockInfo.timeStart);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TOP);
        PendingIntent pendingIntent = PendingIntent.getActivity(context, 1, intent, FLAG_UPDATE_CURRENT);
        String contentTitle = context.getString(R.string.block_canary_class_has_blocked, blockInfo.timeStart);
        String contentText = context.getString(R.string.block_canary_notification_message);
        // 构建 intent 和 pendingIntent，用于发通知
        show(context, contentTitle, contentText, pendingIntent);
    }

    @TargetApi(HONEYCOMB)
    private void show(Context context, String contentTitle, String contentText, PendingIntent pendingIntent) {
        NotificationManager notificationManager = (NotificationManager)
                context.getSystemService(Context.NOTIFICATION_SERVICE);

        Notification notification;
        // 构建通知
        if (SDK_INT < HONEYCOMB) {
            notification = new Notification();
            notification.icon = R.drawable.block_canary_notification;
            notification.when = System.currentTimeMillis();
            notification.flags |= Notification.FLAG_AUTO_CANCEL;
            notification.defaults = Notification.DEFAULT_SOUND;
            try {
                Method deprecatedMethod = notification.getClass().getMethod("setLatestEventInfo", Context.class, CharSequence.class, CharSequence.class, PendingIntent.class);
                deprecatedMethod.invoke(notification, context, contentTitle, contentText, pendingIntent);
            } catch (NoSuchMethodException | IllegalAccessException | IllegalArgumentException
                    | InvocationTargetException e) {
                Log.w(TAG, "Method not found", e);
            }
        } else {
            Notification.Builder builder = new Notification.Builder(context)
                    .setSmallIcon(R.drawable.block_canary_notification)
                    .setWhen(System.currentTimeMillis())
                    .setContentTitle(contentTitle)
                    .setContentText(contentText)
                    .setAutoCancel(true)
                    .setContentIntent(pendingIntent)
                    .setDefaults(Notification.DEFAULT_SOUND);
            if (SDK_INT < JELLY_BEAN) {
                notification = builder.getNotification();
            } else {
                notification = builder.build();
            }
        }
        // 发送通知
        notificationManager.notify(0xDEAFBEEF, notification);
    }
```

## 优缺点分析

1，Looper 中调用 Printer 的 print 函数，有拼接字符串操作，会产生大量的 StringBuilder 对象；影响性能。

2，堆栈记录，捕捉到的方法，可能不是真正耗时的方法；真正耗时的方法，可能已经执行完并在开始捕前出栈；

## 定位不准确的例子

即便不准确，但也能获取有用信息，再根据经验判断。

比如处理一个 msg 前，
post 一个延迟 800 毫秒的记录堆栈 runnable，
处理 msg 的时候，调用了 a，b 两个函数，
a 函数 780 毫秒执行完毕出栈了，
b 函数开始执行 20 毫秒后，
总时间累计到 800 毫秒，记录堆栈 runnable 启动，从函数 b 开始记录。
这种情况，就没有记录到真正耗时的堆栈。

但是我们在跟踪到 b 函数时，可以看看之前都调用了啥函数。像这种 a 函数正好在 b 函数前的，还算好找，如果相隔7，8个函数，那就不容易找了。

## 其他性能检测方案

### Choreography

提交一个回调，每次绘制都会调用。
以此计算两次绘制的耗时，以此统计帧率。
但不能知道哪个方法卡。
BlockCanary 可以知道。

### Aspectjx

[HujiangTechnology/gradle_plugin_android_aspectjx](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)

AOP，面向切面的框架，可以像指定的类插入代码。
缺点：会生成额外的代码。

### ASM

这是一个比 aspectjx 更加底层的方式；
通过操作字节码的方式，不会产生额外的代码，增加了方法数。

### TraceCanary

微信团队出品的，正经玩意！

参考1：[微信自研APM利器Matrix 卡顿分析工具之Trace Canary](https://www.sohu.com/a/314576167_671494) ，引用：
> Matrix 是微信开源的一套完整的 APM 解决方案，内部包含 Resource Canary(资源监测)/Trace Canary(卡顿监测)/IO Canary(IO监测) 等。
> 
> Matrix 内容概览

![](/img/media/16214397028030/16232613757142.png)

原来！Matrix 包含这么多功能！！

参考2：[Matrix-TraceCanary的设计和原理分析手册](https://linjiang.tech/2019/11/12/matrix-trace-canary/)

> **核心组件**

> **LooperMonitor**

> 向主线程的Looper注册MessageLogging和IdleHandler，分别对主线程消息模型 处理每条message 和 消息队列空闲时 进行监听并分发每条message处理的开始和结束事件。

> **UIThreadMonitor**

> TracePlugin模块的核心实现。通过LooperMonitor和Choreographer实现对线程所有UI操作的监控，为具体Tracer提供相应事件的回调和统计。

> **如何实现对UI操作 帧级别 的耗时统计？**

> **通过 LooperMonitor 观察主线程 Handler 的每条 message 的回调事件，并在每次事件结束时向 Choreographer 分别插入 INPUT、ANIMATION、TRAVERSAL 三个事件到对应的队列头部**（实际是先插入INPUT，当INPUT执行时再插入ANIMATION和TRAVERSAL），当下次VSYNC到来时触发执行相应回调，以达到对该三个事件执行时间的精确统计。
> 注意：这里的帧定义为系统处理输入、动画以及绘制的总操作，注意和Handler其余message的区别。由于每一帧执行也是通过Handler触发，所以不管是帧的执行还是其余message，只要出现了耗时，都会导致帧未及时执行或执行过长从而引起卡顿。

> **为什么插入到队列头部？**

> VSYNC事件触发UI相关的操作实际也是通过主线程Handler来完成的，最终执行点是doFrame方法，该方法里是按照INPUT、ANIMATION、TRAVERSAL三种事件依次回调的，从每个事件的队列中取出符合执行时间的集合并执行；这里的问题就是符合时间的事件是取决于doFrame开始执行的时间戳，没法提前知道，即无法成对地向队列尾部插入事件。
> 因此考虑另一种方案：从INPUT队列第一个事件开始执行到ANIMATION队列的第一个事件开始执行刚好就是此次所有INPUT事件的执行时间，以此类推即可计算出三种事件的耗时；但是TRAVERSAL之后就结束了，TRAVERSAL的耗时没法在此次doFrame事件的回调中完成统计，所以我们增加一个标识符，在下一个message事件开始时根据标识符来结束对TRAVERSAL的耗时统计。