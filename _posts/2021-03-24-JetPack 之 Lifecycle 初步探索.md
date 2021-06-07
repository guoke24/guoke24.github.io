---
layout:     post  
title:      "JetPack 之 Lifecycle 初步探索"  
subtitle:   "帮你管理生命周期"  
date:       2021-03-24 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - JetPack  

---

[官网指南](https://developer.android.com/topic/libraries/architecture/lifecycle#kotlin)

## 前言

> 生命周期感知型组件可执行操作来响应另一个组件（如 Activity 和 Fragment）的生命周期状态的变化。这些组件有助于您编写出更有条理且往往更精简的代码，此类代码更易于维护。

> 一种常见的模式是在 Activity 和 Fragment 的生命周期方法中实现依赖组件的操作。但是，这种模式会导致代码条理性很差而且会扩散错误。通过使用生命周期感知型组件，您可以将依赖组件的代码从生命周期方法移入组件本身中。

> androidx.lifecycle 软件包提供了可用于构建生命周期感知型组件的类和接口 - 这些组件可以根据 Activity 或 Fragment 的当前生命周期状态自动调整其行为。

> 在 Android 框架中定义的大多数应用组件都存在生命周期。生命周期由操作系统或进程中运行的框架代码管理。它们是 Android 工作原理的核心，应用必须遵循它们。如果不这样做，可能会引发内存泄漏甚至应用崩溃。

## 正文

### Lifecycle 架构

#### LifecycleOwner

ComponentActivity 实现了 LifecycleOwner 接口；

```
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

#### ComponentActivity

```
// ComponentActivity implement LifecycleOwner

    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
```

getLifecycle 函数，返回的是一个 LifecycleRegistry 实例。
也就是说，ComponentActivity 持有一个 LifecycleRegistry 实例。
LifecycleRegistry 是 Lifecycle 的子类，内部的 mObserverMap 字段，保存观察者；
内部的 mLifecycleOwner 字段，引用着 ComponentActivity。

#### Lifecycle

```
public abstract class Lifecycle {
    
    ...
    
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);
    
    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    
    ...
```

#### LifecycleRegistry

```
public class LifecycleRegistry extends Lifecycle {

    ...
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        // 类型为 LifecycleObserver 的 observer，被封装到 ObserverWithState 对象中，
        // 
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        // 观察者 observer 和 statefulObserver 保存一起保存到 mObserverMap 中 
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }
    
    @Override
    public void removeObserver(@NonNull LifecycleObserver observer) {
        // we consciously decided not to send destruction events here in opposition to addObserver.
        // Our reasons for that:
        // 1. These events haven't yet happened at all. In contrast to events in addObservers, that
        // actually occurred but earlier.
        // 2. There are cases when removeObserver happens as a consequence of some kind of fatal
        // event. If removeObserver method sends destruction events, then a clean up routine becomes
        // more cumbersome. More specific example of that is: your LifecycleObserver listens for
        // a web connection, in the usual routine in OnStop method you report to a server that a
        // session has just ended and you close the connection. Now let's assume now that you
        // lost an internet and as a result you removed this observer. If you get destruction
        // events in removeObserver, you should have a special case in your onStop method that
        // checks if your web connection died and you shouldn't try to report anything to a server.
        // 从 map 中移除观察者
        mObserverMap.remove(observer);
    }
    
    ...
```

LifecycleRegistry 实现了 Lifecycle 中的抽象函数 
addObserver(@NonNull LifecycleObserver observer)、
removeObserver(@NonNull LifecycleObserver observer) ，
把观察者 LifecycleObserver 保存到 mObserverMap 中。

这里有个小细节，即新建了一个 ObserverWithState 对象跟 LifecycleObserver 对象绑定；在接收生命周期事件通知的时候，会先调用其 ObserverWithState 的 dispatchEvent 方法，进而再通知到 LifecycleObserver 的 onStateChanged 方法。具体阐述见下面的“小细节”标题。

#### LifecycleObserver

LifecycleObserver 的定义如下：

```
/**
 * Marks a class as a LifecycleObserver. It does not have any methods, instead, relies on
 * {@link OnLifecycleEvent} annotated methods.
 * <p>
 * @see Lifecycle Lifecycle - for samples and usage patterns.
 */
@SuppressWarnings("WeakerAccess")
public interface LifecycleObserver {

}
```
正如其注释所说，LifecycleObserver 没有任何方法，而是依靠 OnLifecycleEvent 注解来修饰方法，接收生命周期事件。

#### LifecycleEventObserver

其还有一个子接口 LifecycleEventObserver，带有 onStateChanged 函数：

```
/**
 * Class that can receive any lifecycle change and dispatch it to the receiver.
 * <p>
 * If a class implements both this interface and
 * {@link androidx.lifecycle.DefaultLifecycleObserver}, then
 * methods of {@code DefaultLifecycleObserver} will be called first, and then followed by the call
 * of {@link LifecycleEventObserver#onStateChanged(LifecycleOwner, Lifecycle.Event)}
 * <p>
 * If a class implements this interface and in the same time uses {@link OnLifecycleEvent}, then
 * annotations will be ignored.
 */
public interface LifecycleEventObserver extends LifecycleObserver {
    /**
     * Called when a state transition event happens.
     *
     * @param source The source of the event
     * @param event The event
     */
    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}
```

当 ComponentActivity 的生命周期发生变化时，注入到其内部的 ReportFragment 类实例，也会收到相同的生命周期事件，ReportFragment 类实例会拿到 ComponentActivity 持有的 LifecycleRegistry 实例并调用其 handleLifecycleEvent 方法，告知其生命周期发生了变化。

#### ReportFragment

ReportFragment 在 ComponentActivity 的 onCreate 函数内进行注入，

```
// ComponentActivity

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mSavedStateRegistryController.performRestore(savedInstanceState);
        // 注入一个 Fragment
        ReportFragment.injectIfNeededIn(this);
        if (mContentLayoutId != 0) {
            setContentView(mContentLayoutId);
        }
    }
```

```
// ReportFragment

    public static void injectIfNeededIn(Activity activity) {
        // ProcessLifecycleOwner should always correctly work and some activities may not extend
        // FragmentActivity from support lib, so we use framework fragments for activities
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }
```

其实就是通过 FragmentManager 来添加一个 Fragment。

然后，在 ReportFragment 的各个生命周期函数，都有：

```
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }
```

dispatch 函数，会发送各个生命周期事件，其实现如下：

```
    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
```

最终调用到 LifecycleRegistry 的 handleLifecycleEvent 函数，其实现如下：

```
// LifecycleRegistry

    /**
     * Sets the current state and notifies the observers.
     * <p>
     * Note that if the {@code currentState} is the same state as the last call to this method,
     * calling this method has no effect.
     *
     * @param event The event that was received
     */
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }
    
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                    + "garbage collected. It is too late to change lifecycle state.");
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
```

由 LifecycleRegistry 的 handleLifecycleEvent 方法一直跟踪到 sync 方法，
然后调用 forwardPass 或 backwardPass 方法，生命周期前进或后退。

```
    private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
                mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                Event event = downEvent(observer.mState);
                pushParentState(getStateAfter(event));
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }
```

无论前进或后退，都会调用到 `observer.dispatchEvent(lifecycleOwner, event);` 这一句，进行生命周期事件的发送。

observer 是 ObserverWithState 类，实现如下：

```
    static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```

由 dispatchEvent 方法内可知，最终就会调用到 
`mLifecycleObserver.onStateChanged(owner, event);` 这一句，即 LifecycleEventObserver 的 onStateChanged 方法会收到生命周期事件。

#### 小细节

这里有个小细节，在 LifecycleRegistry 的 addObserver 函数中，新建了一个 ObserverWithState 对象跟 LifecycleObserver 对象绑定，作为键值对一起存到了 LifecycleRegistry 对象的 mObserverMap 中 ；在接收生命周期事件通知的时候，会调用其 ObserverWithState 对象的 dispatchEvent 方法，进而再通知到 LifecycleObserver 的 onStateChanged 方法。但 LifecycleObserver 是没有 onStateChanged 方法的，在 ObserverWithState 的构造函数内，会把 LifecycleObserver 转换为 LifecycleEventObserver 对象。

```
    static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            // 这是转换函数
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }
        ...
    }
```

```
// Lifecycling.java

    // 入参的 object，就是外部添加进来的 LifecycleObserver 对象
    // 返回的是 LifecycleEventObserver 接口的实现类
    // 实际的返回对象有五种：
    // 1，FullLifecycleObserverAdapter
    // 2，LifecycleEventObserver（外部传进来的就是该实现类）
    // 3，CompositeGeneratedAdaptersObserver
    // 4，SingleGeneratedAdapterObserver
    // 5，ReflectiveGenericLifecycleObserver
    // 总而言之，这五种对象，最终都会把生命周期事件传递给 object，即 LifecycleObserver 对象。
    @NonNull
    static LifecycleEventObserver lifecycleEventObserver(Object object) {
        boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
        boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
        if (isLifecycleEventObserver && isFullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                    (LifecycleEventObserver) object);
        }
        if (isFullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
        }

        if (isLifecycleEventObserver) {
            return (LifecycleEventObserver) object;
        }

        final Class<?> klass = object.getClass();
        // 构造函数的类型
        int type = getObserverConstructorType(klass);
        if (type == GENERATED_CALLBACK) {
            List<Constructor<? extends GeneratedAdapter>> constructors =
                    sClassToAdapters.get(klass);
            // 一个参数的构造函数
            if (constructors.size() == 1) {
                GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                        constructors.get(0), object);
                return new SingleGeneratedAdapterObserver(generatedAdapter);
            }
            GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
            // 多个参数，即参数为数组的构造函数
            for (int i = 0; i < constructors.size(); i++) {
                adapters[i] = createGeneratedAdapter(constructors.get(i), object);
            }
            return new CompositeGeneratedAdaptersObserver(adapters);
        }
        // 反射生成对象，能回调到「生命周期事件的注解」所修饰的方法就因为这个
        return new ReflectiveGenericLifecycleObserver(object);
    }
```

接下来的解析，引用了 [Lifecycle--生命周期感知型组件，源码分析](https://www.jianshu.com/p/5a273e2ab87f) 的解析：

> ReflectiveGenericLifecycleObserver构造函数通过ClassesInfoCache类的单例调用getInfo()获取CallbackInfo对象的实例：

> ```
> ReflectiveGenericLifecycleObserver(Object wrapped) {
>         mWrapped = wrapped;
>         mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
>     }
> ```

> ClassesInfoCache 内部有两个Map，实现了类信息的缓存

> ```
> /**
>  * Reflection is expensive, so we cache information about methods
>  * for {@link ReflectiveGenericLifecycleObserver}, so it can call them,
>  * and for {@link Lifecycling} to determine which observer adapter to use.
>  */
>  //从描述可以看出，反射的代价是昂贵的，所以这里缓存了methods信息
>  
>  //存储当前类的CallbackInfo的信息
>  private final Map<Class, CallbackInfo> mCallbackMap = new HashMap<>();
>  //存储当前类是否有使用了注解的生命周期方法
>  private final Map<Class, Boolean> mHasLifecycleMethods = new HashMap<>();
>  
>  CallbackInfo getInfo(Class klass) {
>         CallbackInfo existing = mCallbackMap.get(klass);
>         //缓存中有，直接返回
>         if (existing != null) {
>             return existing;
>         }
>         //缓存中没有，进行创建
>         existing = createInfo(klass, null);
>         return existing;
>     }

> //通过反射查找被@OnLifecycleEvent注解修饰的方法，然后保存起来
> private CallbackInfo createInfo(Class klass, @Nullable Method[] declaredMethods) {
>         Class superclass = klass.getSuperclass();
>         Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
>         if (superclass != null) {
>             CallbackInfo superInfo = getInfo(superclass);
>             if (superInfo != null) {
>                 handlerToEvent.putAll(superInfo.mHandlerToEvent);
>             }
>         }

>         Class[] interfaces = klass.getInterfaces();
>         for (Class intrfc : interfaces) {
>             for (Map.Entry<MethodReference, Lifecycle.Event> entry : getInfo(
>                     intrfc).mHandlerToEvent.entrySet()) {
>                 verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
>             }
>         }

>         Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
>         boolean hasLifecycleMethods = false;
>         for (Method method : methods) {
>             //获取方法上的@OnLifecycleEvent注解信息
>             OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
>             if (annotation == null) {
>                 continue;
>             }
>             hasLifecycleMethods = true;
>             Class<?>[] params = method.getParameterTypes();
>             int callType = CALL_TYPE_NO_ARG;
>             if (params.length > 0) {
>                 callType = CALL_TYPE_PROVIDER;
>                 if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
>                     throw new IllegalArgumentException(
>                             "invalid parameter type. Must be one and instanceof LifecycleOwner");
>                 }
>             }
>             Lifecycle.Event event = annotation.value();

>             //参数校验
>             if (params.length > 1) {
>                 callType = CALL_TYPE_PROVIDER_WITH_EVENT;
>                 if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
>                     throw new IllegalArgumentException(
>                             "invalid parameter type. second arg must be an event");
>                 }
>                 if (event != Lifecycle.Event.ON_ANY) {
>                     throw new IllegalArgumentException(
>                             "Second arg is supported only for ON_ANY value");
>                 }
>             }
>             if (params.length > 2) {
>                 throw new IllegalArgumentException("cannot have more than 2 params");
>             }
>             MethodReference methodReference = new MethodReference(callType, method);
>             verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
>         }
>         CallbackInfo info = new CallbackInfo(handlerToEvent);
>         mCallbackMap.put(klass, info);
>         mHasLifecycleMethods.put(klass, hasLifecycleMethods);
>         return info;
>     }
> ```

由此，大概明白了为什么 ReflectiveGenericLifecycleObserver 可以回调到 LifecycleObserver 的实现类中的「生命周期事件的注解」所修饰的方法。


### 图片总结

![屏幕快照 2021-06-07 下午7.12.33](/img/屏幕快照 2021-06-07 下午7.12.33.png)

### 使用总结

参考 [官网指南](https://developer.android.com/topic/libraries/architecture/lifecycle#kotlin) 中：

```
class MyObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun connectListener() {
        ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun disconnectListener() {
        ...
    }
}

myLifecycleOwner.getLifecycle().addObserver(MyObserver())
```

实现 LifecycleObserver 接口，通过注解的方式，指明要接收的生命周期事件类型。
除此之外，还可以通过实现 LifecycleEventObserver 接口的 onStateChanged 方法来接收所有类型的生命周期事件。