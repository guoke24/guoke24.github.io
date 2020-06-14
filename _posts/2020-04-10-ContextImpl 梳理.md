---
layout:     post  
title:      "ContextImpl 梳理"  
subtitle:   "上下文、环境、小组件的容器，大组件的抽象"  
date:       2020-04-10 12:00:00  
author:     "GuoHao"  
header-img: "img/开荒2.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - 四大组件  
    - Context 

---

## 继承关系

引用 [Android深入理解Context（一）Context关联类和Application Context创建过程](http://liuwangshu.cn/framework/context/1-application-context.html) 的一张图：

![](https://s2.ax1x.com/2019/05/28/VejqSI.png)

可知 Activity、Service、Application 都是 ContextWrapper 的子类，
同时也是 Context 子类；<br>
它们都继承了 ContextWrapper 的 mBase 变量，引用着一个 ContextImpl 实例；

引用 [Android深入理解Context（一）Context关联类和Application Context创建过程](http://liuwangshu.cn/framework/context/1-application-context.html) 的文字：


> ContextImpl提供了很多功能，但是外界需要使用并拓展ContextImpl的功能，因此设计上使用了装饰模式，ContextWrapper是装饰类，它对ContextImpl进行包装，ContextWrapper主要是起了方法传递作用，ContextWrapper中几乎所有的方法实现都是调用ContextImpl的相应方法来实现的。
> 
ContextThemeWrapper、Service和Application都继承自ContextWrapper，这样他们都可以通过mBase来使用Context的方法，同时它们也是装饰类，在ContextWrapper的基础上又添加了不同的功能。

> ContextThemeWrapper中包含和主题相关的方法（比如： getTheme方法），因此，需要主题的Activity继承ContextThemeWrapper，而不需要主题的Service则继承ContextWrapper。

<br>

## 提前封装好服务的代理

比如想在 Service 中添加一个 View 到屏幕，需要用到 WindowManager ，

```
WindowManager mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
```
WindowManager 本质是 WindowManagerService 在客户端的间接代理，真正的服务端 WMS 在 SystemServer 中；WindowManager 的实现类 WindowManagerImpl 会委托 WindowManagerGrobal 类，再委托 ViewRootImpl 类，通过 Seesion 与 WMS 进行通信；

为了方便每个进程（APP）跟 WindowManagerService 通信，会有一个类专门实现好 WindowManager，提前静态注册好，每个进程中，都可以快速访问到；

这个专门类是什么？SystemServiceRegistry！

### 与 SystemServiceRegistry 的配合


```
// ContextWrapper.java

    @Override
    public Object getSystemService(String name) {
        return mBase.getSystemService(name);
    }
    
// mBase 就是 ContextImpl.java

    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
    
// SystemServiceRegistry.java

    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
    
    // 在 SystemServiceRegistry.java 中的静态块，会注册很多的服务

    static {
        ...

        registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
                new CachedServiceFetcher<ActivityManager>() {
            @Override
            public ActivityManager createService(ContextImpl ctx) {
                return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
            }});
            
        ...
        
        registerService(Context.WINDOW_SERVICE, WindowManager.class,
                new CachedServiceFetcher<WindowManager>() {
            @Override
            public WindowManager createService(ContextImpl ctx) {
                return new WindowManagerImpl(ctx);
            }});
        
            ...
    }
```

<br>


## ContextImpl 创建时机

Application 和 Activity 创建 ContextImpl 的时序图：

![](/img/创建 ContextImpl 的时序图.png)

<br>
该过程中，

先创建 Activity 的 ContextImpl 实例； by ContextImpl.**createActivityContext**(..) <br>
再创建 Activity 实例；<br>

接着创建 Application 的 ContextImpl 实例；by ContextImpl.**createAppContext**(..) <br>
随后创建 Application 的 实例；然后关联对应的 ContextImpl 实例；<br>
**注意：该步骤一般是在 handleBindApplication 的时候执行的，不会在启动 Activity 之后；**


Activity 实例关联其 ContextImpl 实例；<br>

过程结束。

可见，创建 Activity 的 ContextImpl 实例 和 Application 的 ContextImpl 实例用到 ContextImpl 类中不同的静态方法；

而对于 Service 来说，其对应的 ContextImpl 实例用到 ContextImpl.createAppContext 行法；


```
// ActivityThread.java

    private void handleCreateService(CreateServiceData data) {
        ...

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            // 反射创建一个 Service 实例
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
        } catch ...

        try {
            ...
            // 创建一个 Service 实例对应的 ContextImpl
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            // ContextImpl 关联 Service
            context.setOuterContext(service);
            // 拿到 Application
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            
            // Service 关联 ContextImpl
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            // 开启 service 的生命周期
            service.onCreate();
            // 记录该 service
            mServices.put(data.token, service);
            try {
                // 通知 AMS，已经启动了 service
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch ...
        
    }
```

到此，

### 小结

- Applicaion、Activity、Service 的实例创建，都通过 Factory 来反射创建；

- Applicaion、Activity、Service 三者持有不同的 ContextImpl 实例；

- 关于 ContextImpl 实例的创建：

  - Service 和 Application 一样通过 ContextImpl.createAppContext(..) 方法创建

  - Activity 通过 ContextImpl.createActivityContext(..) 方法创建



### 另外两个 ContextImpl 实例

ActivityThread 还持有另外两个 ContextImpl 实例，如源码：

```
// ActivityThread.java

    private ContextImpl mSystemContext;
    private ContextImpl mSystemUiContext;
    
    public ContextImpl getSystemContext() {
        synchronized (this) {
            if (mSystemContext == null) {
                mSystemContext = ContextImpl.createSystemContext(this);
            }
            return mSystemContext;
        }
    }

    public ContextImpl getSystemUiContext() {
        synchronized (this) {
            if (mSystemUiContext == null) {
                mSystemUiContext = ContextImpl.createSystemUiContext(getSystemContext());
            }
            return mSystemUiContext;
        }
    }

```