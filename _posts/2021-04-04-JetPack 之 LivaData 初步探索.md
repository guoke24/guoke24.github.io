---
layout:     post  
title:      "JetPack 之 LivaData 初步探索"  
subtitle:   "帮你感知生命周期"  
date:       2021-04-04 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - JetPack  

---

## 参考

[Google 参考指南](https://developer.android.com/topic/libraries/architecture/livedata)

[Android 架构组件基本示例](https://github.com/guoke24/architecture-components-samples)

没有专门的 codelab，尴尬。。。

指南里推荐的其他文章：
ViewModel 和 LiveData：模式 + 反模式：
[ViewModels and LiveData: Patterns + AntiPatterns](https://medium.com/androiddevelopers/viewmodels-and-livedata-patterns-antipatterns-21efaef74a54)，看了一半，一知半解；

[ViewModel 之外的 LiveData - 使用 Transformations 和 MediatorLiveData 的响应模式](https://medium.com/androiddevelopers/livedata-beyond-the-viewmodel-reactive-patterns-using-transformations-and-mediatorlivedata-fda520ba00b7)

[LiveData 与信息提示控件、导航和其他事件（SingleLiveEvent 情景）](https://medium.com/androiddevelopers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150)

## 提出你的问题

为何要实例化 MutableLiveData？

观察一个 LiveData，观察者存储到哪里？

通知时，根据观察者的什么属性判断？

## 源码

### 成员变量

```
public abstract class LiveData<T> {
    @SuppressWarnings("WeakerAccess") /* synthetic access */
    final Object mDataLock = new Object();
    
    // 起始版本
    static final int START_VERSION = -1;
    
    // 
    static final Object NOT_SET = new Object();

    private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();

    // how many observers are in active state
    int mActiveCount = 0;
    
    // liveData 持有的真实数据
    private volatile Object mData;
    
    // when setData is called, we set the pending data and actual data swap happens on the main
    // thread
    volatile Object mPendingData = NOT_SET;
    
    // 标识数据的新旧；每调用一次 setData 更新一次
    // 粘性数据会用到
    private int mVersion;

    private boolean mDispatchingValue;
    private boolean mDispatchInvalidated;
    
    // 子线程更新 LiveData 时的任务
    private final Runnable mPostValueRunnable = new Runnable() {
        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            setValue((T) newValue);
        }
    };

```
### 构造函数

```
    /**
     * Creates a LiveData initialized with the given {@code value}.
     *
     * @param value initial value
     */
    public LiveData(T value) {
        mData = value;
        mVersion = START_VERSION + 1;
    }

    /**
     * Creates a LiveData with no value assigned to it.
     */
    public LiveData() {
        mData = NOT_SET;
        mVersion = START_VERSION;
    }
```

### observe(LifecycleOwner,Observer)

```
    /**
     * Adds the given observer to the observers list within the lifespan of the given
     * owner. The events are dispatched on the main thread. If LiveData already has data
     * set, it will be delivered to the observer.
     * 观察者将被添加到观察者列表。在主线程通知观察者。
     * <p>
     * The observer will only receive events if the owner is in {@link Lifecycle.State#STARTED}
     * or {@link Lifecycle.State#RESUMED} state (active).
     * 只有 owner 处于 STARTED 和 RESUMED 状态才会收到通知。
     * <p>
     * If the owner moves to the {@link Lifecycle.State#DESTROYED} state, the observer will
     * automatically be removed.
     * 如果 owner 走到 DESTROYED 状态，将会被移除。
     * <p>
     * When data changes while the {@code owner} is not active, it will not receive any updates.
     * If it becomes active again, it will receive the last available data automatically.
     * 如果观察者从非激活状态有回到激活状态，将会自动收到最后更新的数据通知。
     * <p>
     * LiveData keeps a strong reference to the observer and the owner as long as the
     * given LifecycleOwner is not destroyed. When it is destroyed, LiveData removes references to
     * the observer &amp; the owner.
     * 对 owner 是强引用，destroyed 后被移除。
     * <p>
     * If the given owner is already in {@link Lifecycle.State#DESTROYED} state, LiveData
     * ignores the call.
     * <p>
     * If the given owner, observer tuple is already in the list, the call is ignored.
     * If the observer is already in the list with another owner, LiveData throws an
     * {@link IllegalArgumentException}.
     * 如果 observer 已经存在，并且匹配另一个 owner，该函数抛异常。
     */
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        // 1，包装
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        // 2，存储起来
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        // 3，观察 owner 的生命周期
        owner.getLifecycle().addObserver(wrapper);
    }
```

注释1，对 owner, observer 做一层包装，得到 LifecycleBoundObserver 类型的 wrapper 对象；
注释2，存储 wrapper 对象；如果 observer is already in the list with another owner，抛异常，LiveData 头顶注释有讲到。 存储 wrapper 的是 SafeIterableMap 类型；

注释3，wrapper 对象作为 owner 的生命周期的观察者。

wrapper 对象出现在后续的每一步，
所以接下来看看 LifecycleBoundObserver 类的实现：

#### LifecycleBoundObserver

```
// LiveData.java

class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        @NonNull
        final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }
        
        // 1
        // 这个是 LifecycleEventObserver 接口的方法
        // 能收到 owner 的生命周期变化事件
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            // 当 owner 销毁了，就把自己这个 observer 从 LiveData 中移除
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            // 对 owner 的状态改变，作出响应
            activeStateChanged(shouldBeActive());
        }
        ...
    }
    
    // 再看父类的实现：    
    // LiveData 的观察者抽象的封装类
    private abstract class ObserverWrapper {
        final Observer<? super T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;

        ObserverWrapper(Observer<? super T> observer) {
            mObserver = observer;
        }
        ...
        // 2
        // 这里的 activeState 至的是谁的状态？
        // 本质上指的是 mOwner 的状态是否 isAtLeast(STARTED)
        // 注意，这里是 LiveData 的其中一个 observer，
        // 也就是说，每当有一个 observer 的活跃状态改变，就调用该函数
        
        // 在函数内部通过 LiveData 的 mActiveCount 
        // 判断 LiveData 自身是否还有活跃的观察者
        // 并据此调用 onActive()/onInactive() 函数（子类实现）
        void activeStateChanged(boolean newActive) {
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            mActive = newActive; // LiveData 的 observer 的状态
            
            // 表示在此之前的 LiveData 的活跃状态
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            // LiveData 最新的活跃状态
            LiveData.this.mActiveCount += mActive ? 1 : -1;
            
            // 表示 LiveData 之前不活跃，现在进入活跃状态了
            if (wasInactive && mActive) {
                onActive();
            }
            // LiveData 现在不活跃，本 observer 由活跃变为不活跃状态
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
            // 本 observer 是活跃的，就通知他
            if (mActive) {
                dispatchingValue(this);
            }
        }
    }   
```

每一个 LiveData 的观察者，都被封装到 wrapper 对象内，
其 LifecycleBoundObserver 类实现了 LifecycleEventObserver 接口，
所以说，wrapper 对象内有一个 LiveData 的观察者，
而 wrapper 对象，又作为 owner 的观察者，添加到 owner 的观察者列表中。
简单总结：
> owner.observer -> liveData
> owner.lifecycle <- liveData.wrapper(内有 owner.observer 的引用)

#### 插图

![屏幕快照 2021-05-17 下午4.17.32-w2602](/img/屏幕快照 2021-05-17 下午4.17.32.png)


大概就是这样子。
所以，当 owner 生命周期变化，liveData.wrapper 就会收到通知，
注释1处，wrapper 的 onStateChanged 方法会被调用，执行如下逻辑：
如果 owner 被销毁了，就把自己这个 liveData 的观察者移除，然后返回；
否则调用 wrapper 父类的 activeStateChanged 方法近一步处理，在注释2；

activeStateChanged 函数，先更新 liveData 的观察者的活跃状态，
再更新 liveData 的活跃的观察者的数量，
如果数量从 0 -> 1，就调用 liveData 的 onActive()，表示 liveData 开始活跃；
如果数量从 1 -> 0，就调用 liveData 的 onInactive()，没有观察者，则 liveData 不活跃；
这两个函数都是空实现，留给子类实现的；
最后，如果自身这个观察者是活跃状态，就调用 dispatchingValue 方法，把 mData 通知自身这个观察者；

从这里看，**可提炼两点**：
1，如果一个 owner 重回活跃，即使 liveData 的值没有更新，包含该 owner 的 wrapper 也会调用到 dispatchingValue 方法，wrapper 内的 observer 会收到通知，LiveData 头顶的注释函数写的；

2，如果一个 owner 在订阅前，liveData 的值更新了，该 owner 关联的 observer 在订阅后，是否有可能收到 liveData 的值更新的通知？？？**粘性事件** 是否就是指这个？

关于 dispatchingValue 方法，在 setValue 标题会讲。

接着看看存储 wrapper 的数据结构：

#### SafeIterableMap

该 Map 就是存储

```
// SafeIterableMap.java

    public V putIfAbsent(@NonNull K key, @NonNull V v) {
        Entry<K, V> entry = get(key);
        if (entry != null) {
            return entry.mValue;
        }
        put(key, v);
        return null;
    }
    
    protected Entry<K, V> put(@NonNull K key, @NonNull V v) {
        Entry<K, V> newEntry = new Entry<>(key, v);
        mSize++;
        if (mEnd == null) {
            mStart = newEntry;
            mEnd = mStart;
            return newEntry;
        }

        mEnd.mNext = newEntry;
        newEntry.mPrevious = mEnd;
        mEnd = newEntry;
        return newEntry;

    }
```

SafeIterableMap 本质是一个 LinkedList，其注释写到：

```
/**
 * LinkedList, which pretends to be a map and supports modifications 
 * during iterations.
 * It is NOT thread safe.
 *
 * @param <K> Key type
 * @param <V> Value type
 * @hide
 */
```
可知其以链表的形式存储 map，**支持迭代的时候修改**。@hide 的类。开发者用不到。

到此 observer 的函数看完，接着看看 setValue 和通知逻辑：

### setValue

```
    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;// 1，版本加1
        mData = value;
        dispatchingValue(null);// 2，分发所有
    }
```

### dispatchingValue

```
    void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                // 1，通知一个
                considerNotify(initiator);
                initiator = null;
            } else {
                // 2，通知所有
                for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        // 3，没发生错误，一次循环就出来了，
        // 但如果中途发生错误，mDispatchInvalidated 被置为 true，则继续循环；
        mDispatchingValue = false;
    }
```

### considerNotify

```
    private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        // 1，liveData 和 observer 的版本比较
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        // 2，确保收到通知的 observer，版本跟 liveData 一致
        observer.mLastVersion = mVersion;
        observer.mObserver.onChanged((T) mData);
    }
```

注释1，会出现这种情况：observer 所关联的 owner 进入非活跃状态一段事件，期间 liveData 更新了几次，所以 mVersion > observer.mLastVersion，那么当 owner 重回活跃状态的以后，其关联的 liveData 的 observer 也会收到最近一个 mData 的通知。这个 LiveData 类的头顶注释写有！

## 小结

到此，LiveData 的主要逻辑源码看完。
接着看看「粘性事件」怎么发生，怎么解决。