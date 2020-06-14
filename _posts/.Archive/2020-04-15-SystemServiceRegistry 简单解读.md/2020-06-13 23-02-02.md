---
layout:     post  
title:      "SystemServiceRegistry 简单解读"  
subtitle:   "系统服务注册"  
date:       2020-04-15 12:00:00  
author:     "GuoHao"  
header-img: "img/guitar.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - 四大组件  
    - Context 

---

## Q&A

问：SystemServiceRegistry

答：系统服务注册，其实是注册系统服务在APP本地的代理类，便与快速跟系统服务通信。

---

问：SystemServiceRegistry 这个类是一个进程一个，还是整个系统只有一个？

答：一个进程一个。

---

问：跟 systemserver 有联系吗？

答：有。SystemServiceRegistry 中的静态代码块注册了很多个**服务端的代理类**，方便程序中其他组件通过 ContextImpl 直接获取。

---

## 服务端的代理类保存到哪里


```
    private static final HashMap<Class<?>, String> SYSTEM_SERVICE_NAMES =
            new HashMap<Class<?>, String>();
            
    private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new HashMap<String, ServiceFetcher<?>>();
            
    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }
```

可知保存到 HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS 中，<br>
其中对保存到服务还有一层封装 ServiceFetcher<T>；

引用 [系统服务 SystemServiceRegistry](https://blog.csdn.net/weixin_39821531/article/details/88819715) ：

> 一共有3个抽象类实现了此接口：
> 
> CachedServiceFetcher、StaticServiceFetcher、StaticApplicationContextServiceFetcher
> 
> 它们的工作方式是：如果服务已经存在就直接返回，如果不存在就调用createService抽象方法创建一个;

源码中的解析：


```
    /**
     * Base interface for classes that fetch services.
     * These objects must only be created during static initialization.
     */
    static abstract interface ServiceFetcher<T> {
        T getService(ContextImpl ctx);
    }
    
    // --- CachedServiceFetcher 用得最多，七八十个 ----
    
    /**
     * Override this class when the system service constructor needs a
     * ContextImpl and should be cached and retained by that context.
     */
    static abstract class CachedServiceFetcher<T> implements ServiceFetcher<T> {...}
    
    // --- StaticServiceFetcher 用得第二多，十多个 ----
    
    /**
     * Override this class when the system service does not need a ContextImpl
     * and should be cached and retained process-wide.
     */
    static abstract class StaticServiceFetcher<T> implements ServiceFetcher<T> {...}
    
    // --- StaticApplicationContextServiceFetcher 用得最少，就一个 ----
    
    /**
     * Like StaticServiceFetcher, creates only one instance of the service per application, but when
     * creating the service for the first time, passes it the application context of the creating
     * application.
     *
     * TODO: Delete this once its only user (ConnectivityManager) is known to work well in the
     * case where multiple application components each have their own ConnectivityManager object.
     */
    static abstract class StaticApplicationContextServiceFetcher<T> implements ServiceFetcher<T> {...}
```

略去细节，从目的上看，这样做就是为了节省内存，提高性能吧。。。

因为内创建一个服务的代理类实例，所需内存估计会比较多；

如果把服务的代理类封装到一个 Fetcher 类中，并定义创建该代理类的方法；<br>等到实际需要时（被其他组件调用获取时再创建服务的代理类实例）；<br>
这样在 SystemServiceRegistry 刚被加载时，不会立刻创建 70 多个服务的代理类，而只是 70多个 Fetcher 类实例；

[android笔记之SystemServiceRegistry](https://www.jianshu.com/p/db3ced6c933b) 有稍微详细一点的解析。。。

## 静态代码块的服务

SystemServiceRegistry 在静态代码块注册的服务，大概有70多个；

其中有两种形式：

- 直接拿到服务的代理类
- 间接拿到服务的代理类

见源码：

```
    static {
    
        // AMS 间接代理
        // 内部拿到了 AMS 的代理类
        registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
                new CachedServiceFetcher<ActivityManager>() {
            @Override
            public ActivityManager createService(ContextImpl ctx) {
                // 没有直接 ServiceManager.getService
                // 但其内部还是会调用的
                return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
            }});

        // WMS 间接代理
        // 还得委托 WindowManagerGrobal 向 WMS 通信
        registerService(Context.WINDOW_SERVICE, WindowManager.class,
                new CachedServiceFetcher<WindowManager>() {
            @Override
            public WindowManager createService(ContextImpl ctx) {
                return new WindowManagerImpl(ctx);
            }});
            
        ...

        // 通过 ServiceManager.getService
        // 直接获取 AlarmManagerService 的代理类
        registerService(Context.ALARM_SERVICE, AlarmManager.class,
                new CachedServiceFetcher<AlarmManager>() {
            @Override
            public AlarmManager createService(ContextImpl ctx) throws ServiceNotFoundException {
                // 获取服务端的 binder
                IBinder b = ServiceManager.getServiceOrThrow(Context.ALARM_SERVICE);
                // 转成代理类，相当于客户端
                IAlarmManager service = IAlarmManager.Stub.asInterface(b);
                // 多一层封装，保存返回
                return new AlarmManager(service, ctx);
            }});
    
        ...
    
        // 通过 ServiceManager.getService
        // 直接获取 WallpaperManagerService 的代理类
        registerService(Context.WALLPAPER_SERVICE, WallpaperManager.class,
                new CachedServiceFetcher<WallpaperManager>() {
            @Override
            public WallpaperManager createService(ContextImpl ctx)
                    throws ServiceNotFoundException {
                final IBinder b;
                // 获取服务端的 binder，区分版本
                if (ctx.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.P) {
                    b = ServiceManager.getServiceOrThrow(Context.WALLPAPER_SERVICE);
                } else {
                    b = ServiceManager.getService(Context.WALLPAPER_SERVICE);
                }
                // 转成代理类，相当于客户端
                IWallpaperManager service = IWallpaperManager.Stub.asInterface(b);
                // 多一层封装，保存返回
                return new WallpaperManager(service, ctx.getOuterContext(),
                        ctx.mMainThread.getHandler());
            }});
        
        ...
        
```

上述的 AMS，对应的代理类在 ActivityManager 类的内部，看源码：

```
// ActivityManager 类

    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    // 真正的 AMS 代理类在此
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
            
```

## 小结，提前封装，快速方便

本质上，想要获取 SystemServer 端的服务的代理类，就得通过这两步：

```
    // 获取服务端的 binder
    final IBinder b = ServiceManager.getService(.Context.XYZ_SERVICE.)
    // 转换成服务端的代理类
    IXYZ service = IXYZ.Stub.asInterface(b);
```

SystemServiceRegistry 只是提前做一层封装，再让 APP 中其他组件，通过 Context（ContextImpl） 来获取，快速方便。

## 拓展

[android笔记之SystemServiceRegistry](https://www.jianshu.com/p/db3ced6c933b)

[系统服务 SystemServiceRegistry](https://blog.csdn.net/weixin_39821531/article/details/88819715)
