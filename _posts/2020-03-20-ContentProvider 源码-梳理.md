---
layout:     post  
title:      "ContentProvider 源码-梳理"  
subtitle:   "庞大数据的管理者"  
date:       2020-03-20 12:00:00  
author:     "GuoHao"  
header-img: "img/old-house.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - ContentProvider  
    - 四大组件  

---

## 参考

[Android深入四大组件（五）Content Provider的启动过程 ](http://liuwangshu.cn/framework/component/5-contentprovider-start.html)

## 正文

### 时序图


引用 [Android深入四大组件（五）Content Provider的启动过程 ](http://liuwangshu.cn/framework/component/5-contentprovider-start.html) 的两张图：

<br>

![](https://s2.ax1x.com/2019/05/28/VeHqSK.png)

<br>

![](https://s2.ax1x.com/2019/05/28/VeHHW6.png) 

### 要点提炼

- 由图一可知，query(..) 时如果没有启动 Content Provider，就创建一个

- 图二，在第七个函数 handleBindApplication(..) 中，创建上下文环境 appContext：<br>

```
private void handleBindApplication(AppBindData data) {
 ...
      final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);//1
       try {
              final ClassLoader cl = instrContext.getClassLoader();
              mInstrumentation = (Instrumentation)
                  cl.loadClass(data.instrumentationName.getClassName()).newInstance();//2
          } catch (Exception e) {
           ...
          }
          final ComponentName component = new ComponentName(ii.packageName, ii.name);
          mInstrumentation.init(this, instrContext, appContext, component,
                  data.instrumentationWatcher, data.instrumentationUiAutomationConnection);//3
         ...
          Application app = data.info.makeApplication(data.restrictedBackupMode, null);//4
          mInitialApplication = app;
          if (!data.restrictedBackupMode) {
              if (!ArrayUtils.isEmpty(data.providers)) {
                  installContentProviders(app, data.providers);//5
                  mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
              }
          }
        ...
         mInstrumentation.callApplicationOnCreate(app);//6
        ... 
}
```
注释5，installContentProviders(..) 函数会创建 Content Provider 并调用其 onCreate(..) 函数；<br>
注释6，`mInstrumentation.callApplicationOnCreate(app)` 函数会调用 Application 的 onCreate(..) 函数；<br>

**小结：Content Provider 的 onCreate(..) 在 Application 的 onCreate(..) 前执行** ...

- 图二，在第九个函数 installProvider(..) 中，

```
private IActivityManager.ContentProviderHolder installProvider(Context context,
           IActivityManager.ContentProviderHolder holder, ProviderInfo info,
           boolean noisy, boolean noReleaseNeeded, boolean stable) {
       ContentProvider localProvider = null;
  ...
               final java.lang.ClassLoader cl = c.getClassLoader();
               localProvider = (ContentProvider)cl.
                   loadClass(info.name).newInstance();//1
               provider = localProvider.getIContentProvider();
               if (provider == null) {
                 ...
                   return null;
               }
               if (DEBUG_PROVIDER) Slog.v(
                   TAG, "Instantiating local provider " + info.name);
               localProvider.attachInfo(c, info);//2
           } catch (java.lang.Exception e) {
              ...
               }
               return null;
           }
       }
          ...
       return retHolder;
   }
```

注释1，反射创建 ContentProvider 实例；<br>
注释2，调用 ContentProvider 实例的 attachInfo 函数，并调用其 onCreate() 函数...

```
private void attachInfo(Context context, ProviderInfo info, boolean testing) {
     ...
          ContentProvider.this.onCreate();
      }
  }
```

## 结尾

此次总结比较简单，ContentProvider 的创建流程跟其他三个组件一样，依赖 AMS 进行；其跨进程通信也是依赖 Binder 机制；

## 拓展

老罗的五篇文章：

1、Android应用程序组件Content Provider简要介绍和学习计划

[ http://blog.csdn.net/luoshengyang/article/details/6946067 ](http://blog.csdn.net/luoshengyang/article/details/6946067)


2、Android应用程序组件Content Provider应用实例

[ http://blog.csdn.net/luoshengyang/article/details/6950440 ](http://blog.csdn.net/luoshengyang/article/details/6950440)


3、Android应用程序组件Content Provider的启动过程源代码分析

[ http://blog.csdn.net/luoshengyang/article/details/6963418 ](http://blog.csdn.net/luoshengyang/article/details/6963418)


4、Android应用程序组件Content Provider在应用程序之间共享数据的原理分析

[ http://blog.csdn.net/luoshengyang/article/details/6967204 ](http://blog.csdn.net/luoshengyang/article/details/6967204)


5、Android应用程序组件Content Provider的共享数据更新通知机制分析

[ http://blog.csdn.net/luoshengyang/article/details/6985171 ](http://blog.csdn.net/luoshengyang/article/details/6985171)