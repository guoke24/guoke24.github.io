---
layout:     post  
title:      "JetPack 之 DataBinding 初步探索"  
subtitle:   "帮你绑定视图和数据"  
date:       2021-04-14 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - JetPack  

---

## 前言

官方文档：https://developer.android.com/topic/libraries/data-binding

Google实验室的Demo:：https://github.com/googlecodelabs/android-databinding

官方最佳实践：https://developer.android.com/codelabs/android-databinding#1

主要分析 google 的 databinding 的 codelabs 中 Demo 项目的三个文件：

* [布局文件](https://github.com/googlecodelabs/android-databinding/blob/master/app/src/main/res/layout/solution.xml)

* [Activity 文件](https://github.com/googlecodelabs/android-databinding/blob/master/app/src/main/java/com/example/android/databinding/basicsample/ui/SolutionActivity.kt)

* [ViewModel 文件](https://github.com/googlecodelabs/android-databinding/blob/master/app/src/main/java/com/example/android/databinding/basicsample/data/SimpleViewModelSolution.kt) 

## 基本用法

先看看基本使用：

```
val binding : PlainActivityBinding =
    DataBindingUtil.setContentView(this, R.layout.plain_activity)
```
首先注意名字，binding 的类名是根据 xml 文件名来确定的：
- plain_activity.xml
- Plain + Activity + Binding

plain_activity.xml 布局文件中，有 id 的 view，binding 对象中都有「同名的字段」可以直接访问。这样就省了 findViewById 方法的调用。

plain_activity.xml 布局文件中，通过 `<data>` 和 `<variable>` 标签，可以「嵌入对象」，布局中的 view 的相关属性值，就可以跟这个「嵌入对象」本身（**基本类型**）或者「嵌入对象」的内部属性（**引用类型**）绑定。

然后在 Actiivty 中，**通过改变「嵌入对象」的属性值，就能改变该属性绑定的 View 的属性值**。

更进一步的做法：**用 ViewModel 充当「嵌入对象」**，让 ViewModel 的各个属性绑定布局的 View，我们就可以在 ViewModel 中更改属性的值来改变相应的 View 的属性值。

再更进一步，**ViewModel 的各个属性，可以是 LiveData 类型**，如果两个属性有依赖，联动等关系，可以通过 Transformations.map() 等方法，构建两个 LiveData 的映射关系。

到此，基本认知构建完成。

## 源码探究


```
// Activity

val binding: SolutionBinding =
            DataBindingUtil.setContentView(this, R.layout.solution)
```


```
// DataBindingUtil

    public static <T extends ViewDataBinding> T setContentView(@NonNull Activity activity,
            int layoutId) {
        return setContentView(activity, layoutId, sDefaultComponent);
    }
```


```
// DataBindingUtil

    public static <T extends ViewDataBinding> T setContentView(@NonNull Activity activity,
            int layoutId, @Nullable DataBindingComponent bindingComponent) {
        activity.setContentView(layoutId);
        View decorView = activity.getWindow().getDecorView();
        ViewGroup contentView = (ViewGroup) decorView.findViewById(android.R.id.content);
        return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
    }
```


```
// DataBindingUtil

    private static <T extends ViewDataBinding> T bindToAddedViews(DataBindingComponent component,
            ViewGroup parent, int startChildren, int layoutId) {
        final int endChildren = parent.getChildCount();
        final int childrenAdded = endChildren - startChildren;
        if (childrenAdded == 1) {
            final View childView = parent.getChildAt(endChildren - 1);
            return bind(component, childView, layoutId);
        } else {
            final View[] children = new View[childrenAdded];
            for (int i = 0; i < childrenAdded; i++) {
                children[i] = parent.getChildAt(i + startChildren);
            }
            return bind(component, children, layoutId);
        }
    }
```


```
// DataBindingUtil

    @SuppressWarnings("unchecked")
    static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View[] roots,
            int layoutId) {
        return (T) sMapper.getDataBinder(bindingComponent, roots, layoutId);
    }// roots
    
    static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root,
            int layoutId) {
        return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
    }// 跟上面的区别是 root
    
    private static DataBinderMapper sMapper = new DataBinderMapperImpl();
```

```
// DataBinderMapperImpl.class

    public ViewDataBinding getDataBinder(DataBindingComponent component, View[] views, int layoutId) {
        if (views != null && views.length != 0) {
            int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
            if (localizedLayoutId > 0) {
                Object tag = views[0].getTag();
                if (tag == null) {
                    throw new RuntimeException("view must have a tag");
                }
            }

            return null;
        } else {
            return null;
        }
    }
```

都返回空？？？莫急，编译后生成的。

### 带着问题

问题列表（搞清楚这些问题即可，不必深入透彻研究）：
> 1，生成的辅助类长什么样？
> 2，view 的更新，怎么通知到绑定属性的？
> 3，属性的改动，怎么通知到 view 的？
> 4，为什么说出 Bug，难排查呢？

接着源码追踪：

DataBinderMapperImpl.java 实际是编译生成的类：

```
// build/generated/source/kapt/debug/com/example/android/databinding/basicsample/
// DataBinderMapperImpl.java

  private static final int LAYOUT_PLAINACTIVITYSOLUTION2 = 1;

  private static final int LAYOUT_PLAINACTIVITYSOLUTION3 = 2;

  private static final int LAYOUT_PLAINACTIVITYSOLUTION4 = 3;

  private static final int LAYOUT_PLAINACTIVITYSOLUTION5 = 4;

  private static final int LAYOUT_SOLUTION = 5;

  private static final SparseIntArray INTERNAL_LAYOUT_ID_LOOKUP = new SparseIntArray(5);

  static {
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.example.android.databinding.basicsample.R.layout.plain_activity_solution_2, LAYOUT_PLAINACTIVITYSOLUTION2);
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.example.android.databinding.basicsample.R.layout.plain_activity_solution_3, LAYOUT_PLAINACTIVITYSOLUTION3);
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.example.android.databinding.basicsample.R.layout.plain_activity_solution_4, LAYOUT_PLAINACTIVITYSOLUTION4);
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.example.android.databinding.basicsample.R.layout.plain_activity_solution_5, LAYOUT_PLAINACTIVITYSOLUTION5);
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.example.android.databinding.basicsample.R.layout.solution, LAYOUT_SOLUTION);
  }

```

可见 static 代码块，把 5 个布局添加到 SparseIntArray 集合中，key 是 int 型的代号。

接着看后续：

```
  @Override
  public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
      // 1
      final Object tag = view.getTag();
      if(tag == null) {
        throw new RuntimeException("view must have a tag");
      }
      switch(localizedLayoutId) {
        case  LAYOUT_PLAINACTIVITYSOLUTION2: {
          if ("layout/plain_activity_solution_2_0".equals(tag)) {
            return new PlainActivitySolution2BindingImpl(component, view);
          }
          throw new IllegalArgumentException("The tag for plain_activity_solution_2 is invalid. Received: " + tag);
        }
        ...
        case  LAYOUT_SOLUTION: {
          // 2
          if ("layout/solution_0".equals(tag)) {
            return new SolutionBindingImpl(component, view);
          }
          throw new IllegalArgumentException("The tag for solution is invalid. Received: " + tag);
        }
      }
    }
    return null;
  }
```

注释1处，获取 view 的 tag，后续再注释2中做判断，返回的 SolutionBindingImpl 对象对应了布局 R.layout.solution。

到此调用链基本走通，接下来围绕上述问题进行源码阅读。

### 如何开始绑定

SolutionBindingImpl 的父类是 androidx/databinding/ViewDataBinding.java，
其 OnStartListener 类型的回调接口字段 mOnStartListener，一般在 Activity 的 onStart 事件之前被设置，然后在 Activity 生命周期的 onStart 的时候回调，

#### OnStartListener

```
// ViewDataBinding.java

    static class OnStartListener implements LifecycleObserver {
        final WeakReference<ViewDataBinding> mBinding;
        private OnStartListener(ViewDataBinding binding) {
            mBinding = new WeakReference<>(binding);
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_START)
        public void onStart() {
            ViewDataBinding dataBinding = mBinding.get();
            if (dataBinding != null) {
                // 其实调用了自己的该方法
                dataBinding.executePendingBindings();
            }
        }
    }

    public void executePendingBindings() {
        if (mContainingBinding == null) {
            executeBindingsInternal();
        } else {
            mContainingBinding.executePendingBindings();
        }
    }
    // 最终调用到 executeBindings()
    // 抽象函数，具体实现在子类
    protected abstract void executeBindings();
```

子类 SolutionBindingImpl 类中的 executeBindings() 方法：

#### executeBindings

```
    @Override
    protected void executeBindings() {
        long dirtyFlags = 0;
        synchronized(this) {
            dirtyFlags = mDirtyFlags;
            mDirtyFlags = 0;
        }
        ...

        // 1
        if ((dirtyFlags & 0x3fL) != 0) {
            // 2
            if ((dirtyFlags & 0x31L) != 0) {

                    if (viewmodel != null) {
                        // read viewmodel.lastName
                        viewmodelLastName = viewmodel.getLastName();
                    }
                    // 3
                    updateLiveDataRegistration(0, viewmodelLastName);

                    if (viewmodelLastName != null) {
                        // read viewmodel.lastName.getValue()
                        viewmodelLastNameGetValue = viewmodelLastName.getValue();
                    }
            }
            ...
        }
        // 上边的是给属性设置观察者，这里的属性是 LiveData，观察 LiveData 的方法正是 LivaData 的特性之一。
        
        // 下边是给 view 设置观察者和更新 view 的属性的逻辑，比如：ClickListener
        // batch finished
        if ((dirtyFlags & 0x38L) != 0) {
            // api target 1

            com.example.android.databinding.basicsample.util.BindingAdaptersKt.popularityIcon(this.imageView, viewmodelPopularityGetValue);
            com.example.android.databinding.basicsample.util.BindingAdaptersKt.tintPopularity(this.progressBar, viewmodelPopularityGetValue);
        }
        if ((dirtyFlags & 0x31L) != 0) {
            // api target 1

            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.lastname, viewmodelLastNameGetValue);
        }
        if ((dirtyFlags & 0x20L) != 0) {
            // api target 1
            // 4
            this.likeButton.setOnClickListener(mCallback2);
        }
        if ((dirtyFlags & 0x32L) != 0) {
            // api target 1

            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.likes, integerToStringViewmodelLikes);
            com.example.android.databinding.basicsample.util.BindingAdaptersKt.hideIfZero(this.progressBar, androidxDatabindingViewDataBindingSafeUnboxViewmodelLikesGetValue);
            com.example.android.databinding.basicsample.util.BindingAdaptersKt.setProgress(this.progressBar, androidxDatabindingViewDataBindingSafeUnboxViewmodelLikesGetValue, 100);
        }
        if ((dirtyFlags & 0x34L) != 0) {
            // api target 1

            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.name, viewmodelNameGetValue);
        }
```

注释1、2处，会对 dirtyFlags 标志为判断，这个标致位出现在很多地方，用于标识多种情况。
注释3处，updateLiveDataRegistration 方法，其实就是对 LiveData 添加观察者；
注释4处，是对 view（其实是 button） 添加 click 的观察者；
这样下来，无论是 LiveData 属性改变，还是 Button 被点击，SolutionBindingImpl 都能收到通知，然后作出更新操作。

从布局文件 solution.xml 中发现，只有 id 为 like_button 的 Button，可以接收用户的操作，所以 SolutionBindingImpl 中只对该 Button 设置监听。

到此，怎么设置双向绑定的整体思路就清楚了。可以回答上述的前三个问题了，问题回顾：

插入问题（搞清楚这些问题即可，不必深入透彻研究）：
> 1，生成的辅助类长什么样？
> 2，view 的更新，怎么通知到绑定属性的？
> 3，属性的改动，怎么通知到 view 的？
> 4，为什么说出 Bug，难排查呢？

### 问题回答

1，辅助类是 SolutionBindingImpl.java，它的工作就是帮助建立双向绑定，这部分代码在 executeBindings() 方法中。

2，view 的更新，指的是 view 接收到了用户的操作（点击长按拖动等），在 SolutionBindingImpl # executeBindings() 函数中，会为 view 设置 click listener 等监听，最终把改动传递给 SolutionBindingImpl 处理。

3，属性的改动，比如这里是 LiveData，本身就带有可被观察特性，在 SolutionBindingImpl # executeBindings() 函数中为其设置观察者，最终把改动传递给 SolutionBindingImpl 处理。

注意！若属性是普通的基本类型，不带有 LiveData 的通知能力，
则需要主动调用 binding 的 **invalidateAll() 函数主动刷新**。

4，这个问题暂时无法回答。

接下来可以看看具体实现。

### 如何监听属性的改变

上述注释3处，SolutionBindingImpl 类中的 executeBindings() 方法中调用的 updateLiveDataRegistration 方法，调用的是父类的函数：

#### 发起属性的监听注册

```
// androidx/databinding/ViewDataBinding.java

    protected boolean updateLiveDataRegistration(int localFieldId, LiveData<?> observable) {
        mInLiveDataRegisterObserver = true;
        try {
            // observable 就是一个 ViewModel 中的 LiveData，
            // CREATE_LIVE_DATA_LISTENER 是一个回调接口的构造器
            return updateRegistration(localFieldId, observable, CREATE_LIVE_DATA_LISTENER);
        } finally {
            mInLiveDataRegisterObserver = false;
        }
    }

    private boolean updateRegistration(int localFieldId, Object observable,
            CreateWeakListener listenerCreator) {
        if (observable == null) {
            return unregisterFrom(localFieldId);
        }
        WeakListener listener = mLocalFieldObservers[localFieldId];
        if (listener == null) {
            registerTo(localFieldId, observable, listenerCreator);
            return true;
        }
        // 已经设置了相同，就不再设置
        if (listener.getTarget() == observable) {
            return false;//nothing to do, same object
        }
        unregisterFrom(localFieldId);
        registerTo(localFieldId, observable, listenerCreator);
        return true;
    }
    
    protected void registerTo(int localFieldId, Object observable,
            CreateWeakListener listenerCreator) {
        if (observable == null) {
            return;
        }
        WeakListener listener = mLocalFieldObservers[localFieldId];
        if (listener == null) {
            // 1，利用构造器构造了 WeakListener 对象
            listener = listenerCreator.create(this, localFieldId);
            mLocalFieldObservers[localFieldId] = listener;
            if (mLifecycleOwner != null) {
                listener.setLifecycleOwner(mLifecycleOwner);
            }
        }
        // 调用回调接口，回传 observable（LiveData）
        listener.setTarget(observable);
    }
```

接着看看 listener 是谁？
是 CREATE_LIVE_DATA_LISTENER 构建的 LiveDataListener 的 
getListener() 方法返回的 WeakListener 对象；
带着问题：setTarget 方法的实现是什么？

```
// androidx/databinding/ViewDataBinding.java

    private static final CreateWeakListener CREATE_LIVE_DATA_LISTENER = new CreateWeakListener() {
        @Override
        public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
            // 看，构建的是 LiveDataListener 对象
            // 但返回的是其内部的 Listener 对象
            return new LiveDataListener(viewDataBinding, localFieldId).getListener();
        }
    };
```

##### LiveDataListener

```
// androidx/databinding/ViewDataBinding.java

    private static class LiveDataListener implements Observer,
            ObservableReference<LiveData<?>> {
        final WeakListener<LiveData<?>> mListener;
        LifecycleOwner mLifecycleOwner;

        public LiveDataListener(ViewDataBinding binder, int localFieldId) {
            // 1，setTarget 方法实现在这里
            mListener = new WeakListener(binder, localFieldId, this);
        }

        @Override
        public void setLifecycleOwner(LifecycleOwner lifecycleOwner) {
            LifecycleOwner owner = (LifecycleOwner) lifecycleOwner;
            LiveData<?> liveData = mListener.getTarget();
            if (liveData != null) {
                if (mLifecycleOwner != null) {
                    liveData.removeObserver(this);
                }
                if (lifecycleOwner != null) {
                    liveData.observe(owner, this);
                }
            }
            mLifecycleOwner = owner;
        }

        @Override
        public WeakListener<LiveData<?>> getListener() {
            return mListener;
        }

        @Override
        public void addListener(LiveData<?> target) {
            if (mLifecycleOwner != null) {
                // 2，WeakListener 的 setTarget 方法会调用到这里
                target.observe(mLifecycleOwner, this);
            }
        }

        @Override
        public void removeListener(LiveData<?> target) {
            target.removeObserver(this);
        }

        @Override
        public void onChanged(@Nullable Object o) {
            ViewDataBinding binder = mListener.getBinder();
            binder.handleFieldChange(mListener.mLocalFieldId, mListener.getTarget(), 0);
        }
    }
```

接着看 WeakListener 的源码：

##### WeakListener

```
    private static class WeakListener<T> extends WeakReference<ViewDataBinding> {
        private final ObservableReference<T> mObservable;
        protected final int mLocalFieldId;
        private T mTarget;

        // 1 构造函数，看其调用的地方，传进来的 observable 是 LiveDataListener
        public WeakListener(ViewDataBinding binder, int localFieldId,
                ObservableReference<T> observable) {
            super(binder, sReferenceQueue);
            mLocalFieldId = localFieldId;
            mObservable = observable;
        }

        public void setLifecycleOwner(LifecycleOwner lifecycleOwner) {
            mObservable.setLifecycleOwner(lifecycleOwner);
        }

        // 2，有 1 处可知，mObservable 是 LiveDataListener
        public void setTarget(T object) {
            unregister();
            mTarget = object;
            if (mTarget != null) {
                // 3，由 2 可知，调用的是 LiveDataListener # addListener
                mObservable.addListener(mTarget);
            }
        }

        public boolean unregister() {
            boolean unregistered = false;
            if (mTarget != null) {
                mObservable.removeListener(mTarget);
                unregistered = true;
            }
            mTarget = null;
            return unregistered;
        }

        public T getTarget() {
            return mTarget;
        }

        protected ViewDataBinding getBinder() {
            ViewDataBinding binder = get();
            if (binder == null) {
                unregister(); // The binder is dead
            }
            return binder;
        }
    }
```

绕了半天，最初 SolutionBindingImpl 类中的 executeBindings() 方法中调用的 updateLiveDataRegistration 方法，走到了 LiveDataListener 的 addListener 方法：

```
        @Override
        public void addListener(LiveData<?> target) {
            if (mLifecycleOwner != null) {
                // 其实就是 LivaData 的最常见的设置观察者的调用
                // this 指的是 LiveDataListener 对象
                target.observe(mLifecycleOwner, this);
            }
        }

        // 当 LiveData 有改动，则回调到这里
        @Override
        public void onChanged(@Nullable Object o) {
            ViewDataBinding binder = mListener.getBinder();
            binder.handleFieldChange(mListener.mLocalFieldId, mListener.getTarget(), 0);
        }
```

##### 四个 CREATE

纵观整个 ViewDataBinding，有四个类似 CREATE_LIVE_DATA_LISTENER 的 CREATE：

```
    private static final CreateWeakListener CREATE_PROPERTY_LISTENER = new CreateWeakListener() {
        @Override
        public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
            return new WeakPropertyListener(viewDataBinding, localFieldId).getListener();
            // final WeakListener<Observable> mListener;
        }
    };

    private static final CreateWeakListener CREATE_LIST_LISTENER = new CreateWeakListener() {
        @Override
        public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
            return new WeakListListener(viewDataBinding, localFieldId).getListener();
            // final WeakListener<ObservableList> mListener;
        }
    };

    private static final CreateWeakListener CREATE_MAP_LISTENER = new CreateWeakListener() {
        @Override
        public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
            return new WeakMapListener(viewDataBinding, localFieldId).getListener();
            // final WeakListener<ObservableMap> mListener;
        }
    };

    /**
     * Method object extracted out to attach a listener to a bound LiveData object.
     */
    private static final CreateWeakListener CREATE_LIVE_DATA_LISTENER = new CreateWeakListener() {
        @Override
        public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
            return new LiveDataListener(viewDataBinding, localFieldId).getListener();
            // final WeakListener<LiveData<?>> mListener;
        }
    };
```

这四个 CREATE 内部的 mListener 字段，都持有 WeakListener 对象，getListener() 返回的就是该对象。WeakListener 对象会被传递到 ViewDataBinding # registerTo 方法，该方法内会调用 WeakListener 的 setTarget 方法和 setLifecycleOwner 方法，但最终会调用到这四个 CREATE 内部的 addListener(mTarget) 和 setLifecycleOwner(lifecycleOwner) 方法。

所以，WeakListener 对象就像一个传球手，把注册函数的调用传递给 CREATE。

##### 思考

WeakListener 为何不做成接口，由 CREATE 统一的实现，
而是采用 CREATE 内部持有 WeakListener 对象的方式？

猜想：为了加入「**弱引用**」的特性，和「**泛型**」的支持：

```
private static class WeakListener<T> extends WeakReference<ViewDataBinding>{
        private final ObservableReference<T> mObservable;
        protected final int mLocalFieldId;
        // 泛型，意味着可以监听任意类型的属性
        private T mTarget;

        public WeakListener(ViewDataBinding binder, int localFieldId,
                ObservableReference<T> observable) {
            // 父类的构造函数，传入引用的真实对象
            super(binder, sReferenceQueue);
            mLocalFieldId = localFieldId;
            mObservable = observable;
        }
}
```

那么，为什么要加入「**弱引用**」呢？
回到 ViewDataBinding # registerTo 方法内，创建的 WeakListener 对象，放入到集合中，并关联 localFieldId：

```
// androidx/databinding/ViewDataBinding.java

    protected void registerTo(int localFieldId, Object observable,
            CreateWeakListener listenerCreator) {
        ...
        WeakListener listener = mLocalFieldObservers[localFieldId];
        if (listener == null) {
            // 没有则创建新的 listener 对象，放入 mLocalFieldObservers 集合
            listener = listenerCreator.create(this, localFieldId);
            mLocalFieldObservers[localFieldId] = listener;
            ...
        }
        ...
    }
    
    private WeakListener[] mLocalFieldObservers;
```

再回头看，一个 WeakListener 对象，其实就是对应着一个属性的监听器，而已。
一个 Activity，有一个 xml 布局文件，可以生成一个 binding 对象；
一个 binding 对象，可以有这多个属性绑定多个 view，这里的每个属性，
都会有一个 WeakListener 对象关联。这些个 WeakListener 对象，会放入数组中。

**使用「弱引用」，无非就是为了垃圾回收，避免内存泄漏。**

我的猜想：
首先，WeakListener 和四个 CREATER 的作用，就是给属性设置监听，监听者就是 ViewDataBinding，所以在完成注册任务后，WeakListener 对象就暂时没用了（存疑）；
这种情况采用弱引用，可以在垃圾回收的时候，就回收掉这些 WeakListener 和 CREATER，节省内存，等到下次需要使用的时候再创建。

修正我的猜想：
首先，WeakListener 引用的是 ViewDataBinding 对象，实际就是根据 xml 布局文件生成 SolutionBindingImpl 对象，跟 Activity 是一对一关系，生命周期也同 Activity 一致。

然后，一个布局中，可能会有很多个 view，即很多个属性，那么这每一个属性，都对应一个 WeakListener，所以 WeakListener 对象的数量可以非常多。

如果 Activity 非正常关闭，没有移除这些 WeakListener，
**因为这些都是弱引用，不会阻止 Activity 的回收。**

## ? DataBinding 的生命周期是怎样的



#### 属性改变后怎么回调

##### onChanged

有上述的 LiveDataListener 源码可知，当 LiveData 属性改变后，回调到 LiveDataListener 对象的 onChanged 函数，接着这里看：

```
        @Override
        public void onChanged(@Nullable Object o) {
            ViewDataBinding binder = mListener.getBinder();
            binder.handleFieldChange(mListener.mLocalFieldId, mListener.getTarget(), 0);
        }
```

##### handleFieldChange

```
// ViewDataBinding.java

    private void handleFieldChange(int mLocalFieldId, Object object, int fieldId) {
        if (mInLiveDataRegisterObserver) {
            // We're in LiveData registration, which always results in a field change
            // that we can ignore. The value will be read immediately after anyway, so
            // there is no need to be dirty.
            return;
        }
        // 1
        boolean result = onFieldChange(mLocalFieldId, object, fieldId);
        if (result) {
            // 2
            requestRebind();
        }
    }
```

在注释1处，会设置相应的标志位，然后再注释2的调用中，根据标志位来决定更新哪个 view；

先看注释1的代码跟踪：

##### onFieldChange

```
// ViewDataBinding.java
    protected abstract boolean onFieldChange(int localFieldId, Object object, int fieldId);

    // 子类 SolutionBindingImpl 实现
    @Override
    protected boolean onFieldChange(int localFieldId, Object object, int fieldId) {
        switch (localFieldId) {
            case 0 :
                return onChangeViewmodelLikes((androidx.lifecycle.LiveData<java.lang.Integer>) object, fieldId);
            case 1 :
                return onChangeViewmodelName((androidx.lifecycle.LiveData<java.lang.String>) object, fieldId);
            case 2 :
                return onChangeViewmodelPopularity((androidx.lifecycle.LiveData<com.example.android.databinding.basicsample.data.Popularity>) object, fieldId);
        }
        return false;
    }
    
    private boolean onChangeViewmodelLikes(androidx.lifecycle.LiveData<java.lang.Integer> ViewmodelLikes, int fieldId) {
        if (fieldId == BR._all) {
            synchronized(this) {
                    // 1
                    mDirtyFlags |= 0x1L;
            }
            return true;
        }
        return false;
    }    
```

注释1处设置了 mDirtyFlags 标志位。

再回头看 handleFieldChange 函数源码的注释2，requestRebind() 的调用：

##### requestRebind

```
// ViewDataBinding.java

    protected void requestRebind() {
        if (mContainingBinding != null) {
            mContainingBinding.requestRebind();
        } else {
            final LifecycleOwner owner = this.mLifecycleOwner;
            if (owner != null) {
                Lifecycle.State state = owner.getLifecycle().getCurrentState();
                if (!state.isAtLeast(Lifecycle.State.STARTED)) {
                    return; // wait until lifecycle owner is started
                }
            }
            synchronized (this) {
                if (mPendingRebind) {
                    return;
                }
                mPendingRebind = true;
            }
            if (USE_CHOREOGRAPHER) {
                mChoreographer.postFrameCallback(mFrameCallback);
            } else {
                // 跟踪该行
                mUIThreadHandler.post(mRebindRunnable);
            }
        }
    }
    
    private final Runnable mRebindRunnable = new Runnable() {
        @Override
        public void run() {
            synchronized (this) {
                mPendingRebind = false;
            }
            // 1，清掉无用的弱引用
            processReferenceQueue();

            // 2，不同版本的适配工作
            if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
                // Nested so that we don't get a lint warning in IntelliJ
                if (!mRoot.isAttachedToWindow()) {
                    // Don't execute the pending bindings until the View
                    // is attached again.
                    mRoot.removeOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                    mRoot.addOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                    return;
                }
            }
            // 3，执行绑定
            executePendingBindings();
        }
    };
```

注释1处的源码：

##### processReferenceQueue

```
// ViewDataBinding.java

    private static void processReferenceQueue() {
        Reference<? extends ViewDataBinding> ref;
        while ((ref = sReferenceQueue.poll()) != null) {
            if (ref instanceof WeakListener) {
                WeakListener listener = (WeakListener) ref;
                listener.unregister();
            }
        }
    }
```
可见，是清除已经为回收而入队的弱引用 listener；

再回看上一段注释3的源码调用：

##### executePendingBindings

```
// ViewDataBinding.java

    public void executePendingBindings() {
        if (mContainingBinding == null) {
            executeBindingsInternal();
        } else {
            mContainingBinding.executePendingBindings();
        }
    }
    
    private void executeBindingsInternal() {
        ...
        if (!mRebindHalted) {
            executeBindings();
            if (mRebindCallbacks != null) {
                mRebindCallbacks.notifyCallbacks(this, REBOUND, null);
            }
        }
        ...
    }
    
    protected abstract void executeBindings();
```

可以看到，最终调用的抽象函数 executeBindings()，正是开头的 ViewDataBinding 的子类 SolutionBindingImpl 实现 的 executeBindings() 方法。

简单记忆：
* LiveDataListener # onChanged()
* ViewDataBinding # requestRebind()
* SolutionBindingImpl # executeBindings() 

可见，SolutionBindingImpl 实现 的 executeBindings() 方法，不仅会绑定监听，还会根据 mDirtyFlags 标志位类更新 view。具体代码回看前面的源码。

### 监听 View 的改变

其实就是 OnClickListener 之后的设置，再把通知传给 ViewDataBinding 而已。
源码分析暂时省略。。。

## 小结

到此为止，对 ViewDataBinding 有了一个初步的感性的认知了。
ViewDataBinding 帮你绑定 xml 文件，并为每个 view 创建一个字段，最终封装在一个 binding 对象并返回给你。

binding 对象除了持有 view，也持有与 view 相互绑定的各个属性（常见的用 LiveData 类型）；
其实就是 binding 内部帮忙实现的双向绑定；
无论是 view 还是 view 绑定的属性，都可以通过 binding 访问到，也意味着可以改变它们；
当改变 view 时（UI操作），view 绑定的属性会跟着变；
当改变 view 绑定的属性时，view 也会跟着改变；
你需要做的，是指把原来的 setContentView 改成：

```
val binding : PlainActivityBinding =
    DataBindingUtil.setContentView(this, R.layout.plain_activity)
```

拿到 binding 之后，你就随意改动属性值啦！！！

备注：某些属性，不具备通知能力，需要主动调用 binding 的主动刷新方法：invalidateAll()。

拓展：这种的做法，使得 binding 成为了视图调用的唯一入口，保证了视图调用的唯一性。
听起来都很棒，**但为何。。。用的人。。。不多呢**？

插图一张！！

![屏幕快照 2021-05-11 上午2.41.20](/img/屏幕快照 2021-05-11 上午2.41.20.png)

