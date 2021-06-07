---
layout:     post  
title:      "JetPack 之 Navigation 深度入门"  
subtitle:   "帮你管理 Fragment 的跳转"  
date:       2021-04-25 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - JetPack  

---

## 前言

Navigation 就是针对 Fragment 跳转的一套组件封装。

在 Navigation 之前，fragment 的跳转代码，一般是获取 FragmentManager，再 beginTransaction 操作，拿到 FragmentTransaction 实例后，就可以 add、hide、replace 操作等，最后 commit 操作。为了统一，可能会把这些代码封装到一个类中，作为静态方法。然后再在其他地方调用。

而相关的调用代码，可能分散的出现在各个地方，这样出现的一个情况是：
虽然跳转代码封装到一处，倒是想要全局一览所有的 frgment 的跳转关系，还是比较麻烦。

Navigation 的主要作用，帮你做 fragment 的跳转。
具体做法：把一个 Activity 中的所有 Fragment 的跳转关系，集中配置到一个 xml 文件中，将 Fragment 的跳转集中控制，便于统一管理，强化代码的一致性。

## 相关资料

> 在 Google I/O 2018 上新出现了一个导航组件（Navigation Architecture Component），导航组件类似iOS开发里的StoryBoard，可以可视化的编辑App页面的导航关系。
> 官方文档：[The Navigation Architecture Component](https://developer.android.com/guide/navigation)
> 官方教程：[Navigation Codelab](https://developer.android.com/codelabs/android-navigation#0)
> Google实验室的Demo: [android-navigation](https://github.com/googlecodelabs/android-navigation)

第三方参考文章：[Android架构组件-Navigation的使用(一)](https://www.jianshu.com/p/729375b932fe)

## 使用认知建立

为了对 Navigation 有一个快速的全局的认知，我打算这么说：

Navigation 就是帮你把 fragment 跳转的事情做了！
因此需要引入几个新的类，它们分别是；
- 导航图，一个 xml 文件，在 AndroidStudio 中可以渲染为可视化操作的界面，清晰的显示各个 fragment 之间的跳转关系。
- NavHost：显示导航图中目标的空白容器。导航组件包含一个默认 NavHost 实现 (**NavHostFragment**)，可显示 Fragment 目标。
- NavController：在 NavHost 中管理应用导航的对象。它就是安排 fragment 显示在容器中的角色。开发者可以在 Activity/Fragment 子类中，拿到该实例，从而进行 fragment 的跳转操作。

![屏幕快照 2021-05-12 下午5.12.23](/img/屏幕快照 2021-05-12 下午5.12.23.png)

到此建立全面的感性认知，具体细节，可查看上述链接的文档。。。

## Args 和 Directions 类

在导航图 xml 文件中，每个 `<fragment>` 标签内，
都可以用标签 `<argument>` 给该 fragment 声明接收的参数，
编译器会为其生成一个 Args 类；
**这个 Args 类，帮你从 Bundle 「读」和「写」参数；**
并且，fragment 内部通过 Args 拿到的数据是「类型安全」的。

那么 Directions 类，有什么用？
如果导航图 xml 文件中，有 `<action>` 标签表明两个 fragment 之间会跳转，
比如：A -> B，
编译器会为其会帮你生成一个 ADirections 类；
**这个 ADirections 类，帮你生成一个可以传递给 B fragment 的带参数的 bundle 对象，**
所带的参数，就是在导航图 xml 文件中 B fragment 的 `<argument>` 标签标明的。

当你想要从 A 跳转到 B 时，调用 ADirections 类的 nextAction 函数，传入参数值，
这个参数值，就会被封装到一个 Bundle 中，Bundle 再封装到 NavDirections 对象并返回，
你拿到 NavDirections 对象，调用
 `findNavController().navigate(NavDirections 对象)` 
 就可以跳转并传参了。

## Safe Args 安全在哪里

我认为：安全在于「它帮你读写参数」，你不自己读写，免去写错类型的可能。
> 对 Bundle 读写同一个 key 的参数，如果类型不一致，并不能在编码阶段反馈给你。
> 举例：
> // 调用该行代码
> bundle.getString("plantName")
> // 并不知道，是否有这样代码
> putString("plantName", this.plantName)
> // 也可能是
> putInt("plantName", 1)
// 这个要等到运行时才会报错，也就是「不能在编码阶段反馈」

而 Safe Args 组件，生成的 Directions 类和 Args 类，内部封了一层逻辑：帮你读写 Bundle 参数；
你在使用 Directions 类跳转时，如果传的参数类型不对，就会有IDE的错误提示，即「在编码阶段反馈」。

参考：官方指南：[使用 Safe Args 传递安全的数据](https://developer.android.com/guide/navigation/navigation-pass-data#Safe-args)


