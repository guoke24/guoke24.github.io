---
layout:     post  
title:      "ContentProvider 使用-梳理"  
subtitle:   "庞大数据的管理者"  
date:       2020-03-18 12:00:00  
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

原理优点写得不错，代码样例略微简单：[Android：关于ContentProvider的知识都在这里了！](https://blog.csdn.net/carson_ho/article/details/76101093)

同作者：[使用ContentProvider](https://blog.csdn.net/a992036795/article/details/51610936)，[ContentProvider原理分析](https://blog.csdn.net/a992036795/article/details/51612425)，为例看原理而看其demo

代码样例细节到位，分析在注释中：[ContentObserver的使用完整详细示例 ](https://www.cnblogs.com/xiaoxiaoshen/p/5191186.html)

## 正文

### 设计思路

ContentProvider 作为 四大组件之一，其作用是封装好统一的接口提供给外部，让外部可以通过同意的接口访问某一种类型的数据集。

内部的数据存储的实现，跟外部隔离。

ContentProvider 其实是一个框架，其使用方法属于拓展类套路。
其派生类需要重写6个函数：
初始化函数 onCreate，绑定数据源；
增删改查四个函数，统一的数据操作接口；
类型判断函数 getType，判断某个uri的类型；
其中，增删改查四个函数是提供给外部的接口！

外部通过 ContentResolver 来访问某个 ContentProvider，方式：

```
getContext().getContentResolver().insert( uri，ContentValue )
```

Uri 会指定某个具体的 ContentProvider，通过 ContentProvider 程序的 AndroidManifest.xml 清单文件确定是否匹配，然后该 ContentProvider 的 insert 函数会被调用，在函数内部，借用 UriMatcher 类解析 uri 并执行相应的操作。

Uri 的匹配解析，有一套规则。

ContentObserver 可以监听某个 ContentProvider，属于观察者模式。

<br>

### 要点小结

为了完整体验一把 Content Provider 的使用，需要分别实现 「ContentProvider 端」和「client 端」两个程序，这里简单总结一下，两个端有什么要点：

- ContentProvider 端
    - 明白 ContentProvider 的子类要重写的六个函数在 ContentProvider 框架角度下的意义
    - AndroidManifest.xml 注册 ContentProvider 时各个属性的意义，
        - 其他程序通过 uri 在此处找寻匹配 ContentProvider
    - URI 的语法结构和 UriMatcher 的使用
    - SQLiteOpenHelper 的使用，在 ContentProvider 内部维护数据集的方法就是通过 SQLiteOpenHelper 框架
    - 通知观察者 ContentObserver
        - 每当有增删改操作，调用：
        - getContext().getContentResolver().notifyChange(uri, null);

- client 端
    - ContentResolver 和 ContentValues 相结合去访问 ContentProvider 提供的增删改查接口
    - URI 的使用
    - ContentObserver 的使用
        - ContentObserver 的子类实现 onChange 函数，
        - 通过 ContentResolver 的 registerContentObserver 函数注册
        - 参数指定监听的 uri 对应的 ContentProvider 和 ContentObserver 的子类
    - 增删改这三个函数返回的是改动的行数
    - 查这个函数返回的是一个游标，使用后要关闭

<br>

### 使用方法的 Demo 链接

包含四个工程的demo：

https://github.com/guoke24/ContentProviderDemo

重点看下面：

服务端 Providertest2 ：

[服务端的工程链接](https://github.com/guoke24/ContentProviderDemo/tree/master/Providertest2)

[ContentProvider 的实现：ContentProviderTest.java](https://github.com/guoke24/ContentProviderDemo/blob/master/Providertest2/app/src/main/java/com/guohao/providertest2/ContentProviderTest.java)

[数据库的实现：SQLiteDatabaseOpenHelper.java](https://github.com/guoke24/ContentProviderDemo/blob/master/Providertest2/app/src/main/java/com/guohao/providertest2/SQLiteDatabaseOpenHelper.java)

客户端 Providertest2_client：

[客户端的工程链接](https://github.com/guoke24/ContentProviderDemo/tree/master/Providertest2_client)

[主要在这里：MainActivity.java](https://github.com/guoke24/ContentProviderDemo/blob/master/Providertest2_client/app/src/main/java/com/guohao/providertest2_client/MainActivity.java)

<br>

### 原理

主要参考：[ContentProvider原理分析](https://blog.csdn.net/a992036795/article/details/51612425)

从 ContentResolver 到 ContentProvider，其实也是跨进程的通信，通过 AMS 作为中介来传递信息，跨进程方式是 Binder 机制。

更具体的启动流程，可以参考刘望舒的博客...

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