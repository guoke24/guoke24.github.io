---
layout:     post  
title:      "BroadcastReceiver 源码-梳理"  
subtitle:   "小喇叭人人能听见"  
date:       2020-03-24 12:00:00  
author:     "GuoHao"  
header-img: "img/old-house.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - BroadcastReceiver  
    - 四大组件  

---

## 参考

[Android深入四大组件（四）广播的注册、发送和接收过程](https://blog.csdn.net/itachi85/article/details/71629201)

[独立博客版：Android深入四大组件（四）广播的注册、发送和接收过程 ](http://liuwangshu.cn/framework/component/4-broadcastreceiver.html)

## 正文

图片引用：[独立博客版：Android深入四大组件（四）广播的注册、发送和接收过程 ](http://liuwangshu.cn/framework/component/4-broadcastreceiver.html)

<br>

#### 广播的注册流程时序图

![](https://s2.ax1x.com/2019/05/28/VeA1Ds.png)

<br>

#### 广播的发送、接收流程时序图

![](https://s2.ax1x.com/2019/05/28/VeA3bn.png)

<br>

![](https://s2.ax1x.com/2019/05/28/VeAGEq.png)

<br>

#### 要点提炼

- 由图一可知，发送广播是依赖 AMS 执行的...

- 主要看第三图，AMS 委托 BroadcastQueue.java 处理广播...

- 在第三个函数 processNextBroadcast(...) 中，遍历每个广播并发送给其所有的接收者，函数内一个while循环内嵌套for循环，while循环一个遍历广播，for循环遍历广播的接收者；<br>

- 给每个接收者发送广播时，都调用一次 deliverToRegisteredReceiverLocked(..) 函数...

- 在第四个函数 deliverToRegisteredReceiverLocked(..) 中，检查广播发送者和广播接收者的权限，最后执行发送逻辑，

- 在第五个函数 performReceiveLocked(..) 中，发起跨进程通信，通过调用：

```
    app.thread.scheduleRegisteredReceiver(...)
```

此处的 `app.thread` 是 IApplicationThread，最终执行到<br>
ApplicationThread 类的 scheduleRegisteredReceiver(..) 函数，
ApplicationThread 是 ActivityThread 的内部类...

- 从第六个函数 scheduleRegisteredReceiver(..) 开始，**执行环境切换到 APP**；

- 再看到第八个函数 performReceive(..)，其源码：<br>

```
public void performReceive(Intent intent, int resultCode, String data,
              Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
          final Args args = new Args(intent, resultCode, data, extras, ordered,
                  sticky, sendingUser);//1
          ...
          if (intent == null || !mActivityThread.post(args)) {//2
              if (mRegistered && ordered) {
                  IActivityManager mgr = ActivityManagerNative.getDefault();
                  if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                          "Finishing sync broadcast to " + mReceiver);
                  args.sendFinished(mgr);
              }
          }
      }
```

关键代码：`mActivityThread.post(args)` <br>
注释1，intent 封装到 args 中；<br>
注释2，mActivityThread 是一个 Handler 对象，具体指向的就是 H；<br>
所以 **args 的 run 函数就切换到了 ActivityThread 主线程执行**；<br>
args 的 run 函数源码：<br>

```
public void run() {
  ...
     try {
         final BroadcastReceiver receiver = mReceiver;
         ...
         ClassLoader cl =  mReceiver.getClass().getClassLoader();
         intent.setExtrasClassLoader(cl);
         intent.prepareToEnterProcess();
         setExtrasClassLoader(cl);
         receiver.setPendingResult(this);
         receiver.onReceive(mContext, intent);//1
     } catch (Exception e) {
        ...
     }
    ...
 }
```

可见最终调用了 receiver.onReceive，即广播接收器 BroadcastReceiver 的 onReceive 函数...

<br>

## 结尾

到此源码简单梳理完毕，详情参阅：[Android深入四大组件（四）广播的注册、发送和接收过程 ](http://liuwangshu.cn/framework/component/4-broadcastreceiver.html)