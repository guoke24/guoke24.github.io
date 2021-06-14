---
layout:     post  
title:      "JetPack 之 LiveData 继续探索 | 粘性事件"  
subtitle:   "活的数据，有时也会过于活跃"  
date:       2021-04-30 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - JetPack

---

参考：[重学安卓：LiveData 数据倒灌 背景缘由全貌 独家解析](https://xiaozhuanlan.com/topic/6719328450) 
和对应的 [Github 工程](https://github.com/guoke24/Jetpack-MVVM-Best-Practice) 中的文件 [ProtectedUnPeekLiveData.java]()

在 JetPack 的系列组件中，Lifecycle、ViewModel 组件使用时都感到很方便，很安心，无需重写改造。但是 LiveData 组件，虽然也很方便，但不够安心，因为「粘性事件」的存在。

产生的原因是：添加观察者时，会收到之前的数据通知。
而其中一种解决思路为：在入口和出口处，多家一层判断。

## 入口 | 观察者的注册

既然要加一层判断，根据开闭原则，就得拓展 LiveData。
ProtectedUnPeekLiveData 就是 LiveData 的拓展类。

新增了公开给外部调用的注册方法：

```
// ProtectedUnPeekLiveData

    public void observeInActivity(@NonNull AppCompatActivity activity, @NonNull Observer<? super T> observer) {
        LifecycleOwner owner = activity;
        // 生成一个 hashcode
        Integer storeId = System.identityHashCode(activity.getViewModelStore());
        observe(storeId, owner, observer);
    }

    public void observeInFragment(@NonNull Fragment fragment, @NonNull Observer<? super T> observer) {
        LifecycleOwner owner = fragment.getViewLifecycleOwner();
        Integer storeId = System.identityHashCode(fragment.getViewModelStore());
        observe(storeId, owner, observer);
    }
```

内部私有的核心方法，要加的那一层判断就在这里：

```
// ProtectedUnPeekLiveData

    private final HashMap<Integer, Boolean> observers = new HashMap<>();

    private void observe(@NonNull Integer storeId,
                         @NonNull LifecycleOwner owner,
                         @NonNull Observer<? super T> observer) {

        if (observers.get(storeId) == null) {
            // 更注册的观察者，绑定的 boolean 默认为 true，只有 false 才能收到消息
            observers.put(storeId, true);
        }

        super.observe(owner, t -> {
            if (!observers.get(storeId)) {
                observers.put(storeId, true);
                if (t != null || isAllowNullValue) {
                    // 经过自定义的判断逻辑，再通知真正的观察者
                    observer.onChanged(t);
                }
            }
        });
    }
```

在内部私有的核心方法 observe 中，调用了父类即 LiveData 的 observe，添加的观察者，不是外部传进来的，而是新建的一个匿名的观察者对象；目的在于，收到更新通知时，就可以加入一层自己的判断逻辑，再决定是否通知真正的观察者。

这里添加的判断逻辑，**就是跟观察者绑定的 Boolean 值是否为 false，只有 false，才会通知真正的观察者**。而从入口进来的观察者，绑定的 Boolean 值默认为 true。

与之呼应的，是在出口处，**每通知一个观察者，就把跟观察者绑定的 Boolean 值改为 false**。

这一入一出的呼应，就保证了：**观察者只有在走过一遍「数据更新的出口」的时候，才会真正的收到消息通知**。

## 出口 | 数据的更新通知

```
// ProtectedUnPeekLiveData

    @Override
    protected void setValue(T value) {
        if (value != null || isAllowNullValue) {
            for (Map.Entry<Integer, Boolean> entry : observers.entrySet()) {
                // 每通知一个观察者，就把跟观察者绑定的 Boolean 值改为 false
                // 如此这样，真正的观察者就能收到消息通知了
                entry.setValue(false);
            }
            super.setValue(value);
        }
    }
```

## System.identityHashCode

```
// ProtectedUnPeekLiveData

    public void observeInActivity(@NonNull AppCompatActivity activity, @NonNull Observer<? super T> observer) {
        LifecycleOwner owner = activity;
        // 生成一个 hashcode
        Integer storeId = System.identityHashCode(activity.getViewModelStore());
        observe(storeId, owner, observer);
    }
```

System.identityHashCode 的函数实现：

```
// java/lang/System.java

    /**
     * Returns the same hash code for the given object as
     * would be returned by the default method hashCode(),
     * whether or not the given object's class overrides
     * hashCode().
     * The hash code for the null reference is zero.
     *
     * @param x object for which the hashCode is to be calculated
     * @return  the hashCode
     * @since   JDK1.1
     */
    public static int identityHashCode(Object x) {
        if (x == null) {
            return 0;
        }
        return Object.identityHashCode(x);
    }
```

由此可知，这里生成的 hashcode，是跟 Activity 对象一一对应的。

如果一个 Activity 调用两次 observeInActivity 函数来添加两个观察者，都是传进去 this，那么产生的 hashcode 即 storeId 是相同的，而底层又是依赖 HashMap 存储，所以第一次添加的观察者，会被第二次添加的观察者覆盖吗？不会，因为如果 storeId 存在，就不再添加观察者。

那么，为什么要做「一个 Activity 只能对一个 ProtectedUnPeekLiveData 添加一个观察者」这样的限制呢？

猜想：
是为了「唯一可信源」的理念，保证一对一的关系。
换个角度想，一个 Activity 对每一个 LiveData 只有一个 观察者，
这个规则是有助于保持架构的简洁，功能也够用，维护也方便。

## 小结

ProtectedUnPeekLiveData 是解决粘性事件的一种方案。
ProtectedUnPeekLiveData 有两个特点：
- 1，一个 Activity 对每一个 LiveData 只有一个 观察者
    - 唯一可信源的具体体现。
- 2，出入口加一层判断
    - 添加进来的外部观察者（附加值 true），又包了一层内部匿名观察者内，匿名观察者收到消息变更通知时，判断外部观察者的附加值为 false 才会通知它。这就是所谓入口加判断。
    - setValue 函数发出通知前，会把所有的观察者的附加值设置为 false。这就是所有的出口加判断。
    - 效果：添加进来的外部观察者，没经历 setValue 函数操作，则收不到消息通知，其实就是收不到添加进来这个操作之前的 setValue 函数操作的通知。这就解决了粘性事件。