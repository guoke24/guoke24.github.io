---
layout:     post  
title:      "JetPack 之 DataBinding 继续探索"  
subtitle:   "帮你绑定视图和数据"  
date:       2021-04-20 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - JetPack  

---

## Binding 的命名规则

布局文件 fragment_view_pager.xml，
就会得懂 FragmentViewPagerBinding，
它是 ViewDataBinding 的子类。

## 几种绑定的写法

写法有多种，但本质就一种，核心理念就是 xml 布局文件的名字会被映射成一个驼峰写法的类名。

```
// Activity 中的写法

    // 第一种写法
    DataBindingUtil.setContentView<ActivityGardenBinding>(this, R.layout.activity_garden)

    // 第二种写法，函数调用时不再需要传递类型，因为类型推断！
    //           activity_garden + binding 就是类名
    val binding: ActivityGardenBinding =
            DataBindingUtil.setContentView(this,R.layout.activity_garden)

    // 第三种写法
    val binding = ActivityGardenBinding.inflate(layoutInflater)
    setContentView(binding.root)
```

```
// Fragment 中的写法

// FragmentViewPagerBinding 可以跳转到  xml 布局
//            fragment_view_pager.xml
val binding = FragmentViewPagerBinding.inflate(inflater, container, false)

return binding.root
```

## 生命周期 owner 一定要设置吗

```
    /**
     * Sets the {@link LifecycleOwner} that should be used for observing changes of
     * LiveData in this binding. If a {@link LiveData} is in one of the binding expressions
     * and no LifecycleOwner is set, the LiveData will not be observed and updates to it
     * will not be propagated to the UI.
     *
     * @param lifecycleOwner The LifecycleOwner that should be used for observing changes of
     *                       LiveData in this binding.
     */
    @MainThread
    public void setLifecycleOwner(@Nullable LifecycleOwner lifecycleOwner) {
```

从注释可知，有使用到 LiveData 的时候，才需要设置，
否则，LiveData 就无法感知 Activity 的生命周期变化。

## Binding apply

绑定视图，得到 binding 后，可以配合 apply 来给各个 view 设置相关属性和监听。

## adaptper 相关的一篇英文文章

[Data Binding — lessons learnt](https://medium.com/androiddevelopers/data-binding-lessons-learnt-4fd16576b719#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6ImQzZmZiYjhhZGUwMWJiNGZhMmYyNWNmYjEwOGNjZWI4ODM0MDZkYWMiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2MjA4OTc0OTIsImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjEwOTUxODg0NzA0NTU3MDY4MDExMyIsImVtYWlsIjoiZ3Vva2UyNHNAZ21haWwuY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImF6cCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsIm5hbWUiOiLpg63osaoiLCJwaWN0dXJlIjoiaHR0cHM6Ly9saDMuZ29vZ2xldXNlcmNvbnRlbnQuY29tL2EvQUFUWEFKeE52TjZGQW5lYy1zb2lGUDJ4c3JYWjhlMl9rdm5YLTZHRFQ0X1k9czk2LWMiLCJnaXZlbl9uYW1lIjoi6LGqIiwiZmFtaWx5X25hbWUiOiLpg60iLCJpYXQiOjE2MjA4OTc3OTIsImV4cCI6MTYyMDkwMTM5MiwianRpIjoiNjQ1NGE3MTM3M2E3M2U5YzhmNmU2N2Q4ZmE2NTNhNDU0ZDdiNThlOCJ9.HiSQogW--WsxNpY1pVBRl07ZjnHXCIUiWbbKgM7o7Bzc7toHwEc_kx0GhHnV2Ka6E1MAUWXCHuQoktNdV-1P3L9eIoPC7Rm3ZPZ1UMYAT73ydksFiaDjv1h7gF0SIwh40U7vrDzUXpy4sfDufA6Gl5Q_g5PL9KNZpNiwtjYR9J0_4idTzl9irgEU87HuYfegSWhKJlCZ4xPHjledoHUH89pQYtXc-cYGUD-aVoozgS2X-gQxWRtx6tj9LCB3fNQCXLrxVS5ylkVziDkjua71yMPD72lSAHUhei3kaFJMYiwItTL-aEIYJPZJ8JT-eaePo3GrmPSF5AqoPnrG5HnAFw)