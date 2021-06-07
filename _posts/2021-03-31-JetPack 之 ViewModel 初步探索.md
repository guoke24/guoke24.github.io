---
layout:     post  
title:      "JetPack 之 ViewModel 初步探索"  
subtitle:   "帮你存储界面数据"  
date:       2021-03-31 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - JetPack  

---

## 带着问题

ViewModel 能保存 Activity 因旋转而不能保存的数据。

ViewModel 不会因为 Activity 的销毁而被销毁？

那么 Activity 被销毁而没有被重建的时候，ViewModel 会被销毁咩？

如果是的，那么让 ViewModel 暂时的持有一会 Activity、View 的引用，又有什么风险呢？

带着问题，看看基础使用吧。

## 基础使用

在 Activity 中获取 ViewModel：

```
chronometerViewModel = 
    new ViewModelProvider(this).get(ChronometerViewModel.class);
```

在 Fragment 中获取 Activity 的 ViewModel：

```
mSeekBarViewModel = 
    new ViewModelProvider(requireActivity()).get(SeekBarViewModel.class);
```

Fragment 如何获取独立的 ViewModel？传 this，其实就是传一个 **ViewModelStoreOwner**；

```
mSeekBarViewModel = 
    new ViewModelProvider(this).get(SeekBarViewModel.class);
```

还有别的获取方式吗？

以前的 Java 写法，已经被 @Deprecated

```
mViewModel = ViewModelProviders.of(this).get(ScoreViewModel.class);
```

委托的写法，带有 Factory ，让我懵逼：

```
// google 官方 demo： sunflower app
// PlantDetailFragment.kt
    @Inject
    lateinit var plantDetailViewModelFactory: PlantDetailViewModelFactory

    private val plantDetailViewModel: PlantDetailViewModel by viewModels {
        PlantDetailViewModel.provideFactory(plantDetailViewModelFactory, args.plantId)
    }
```

从 ViewModelProvider 的构造函数看，Factory 确实必须要有的，要么传入一个，要么默认。

接着从源码开始探索。。。

## 源码

### ViewModelProvider(owner)

```
// ViewModelProvider.java

    /**
     * Creates ViewModelProvider. This will create ViewModels
     * and retain them in a store of the given ViewModelStoreOwner.
     * 
     * This method will use the
     * HasDefaultViewModelProviderFactory#getDefaultViewModelProviderFactory() 
     * default factory 
     * if the owner implements HasDefaultViewModelProviderFactory. 
     * Otherwise, a NewInstanceFactory will be used.
     */
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }

    public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }
```

通过该构造函数注释，我们知晓了 ViewModel 和 ViewModelProvider 背后的那些类：
ViewModelStoreOwner、Factory。

ViewModelStoreOwner 就是 Activity 或 Fragment，顾名思义就是 ViewModel存储器的拥有者，也就是说  Activity 或 Fragment 都有存储 ViewModel 的能力。
引申问题：怎么存储的，用什么数据结构呢？

Factory 就是构建 ViewModel 的工厂，为何需要通过 Factory，ViewModelProvider 直接构建不就完事了嘛？哦，可能是考虑到：用户需要自己定义构造逻辑的时候（即需要给构造函数传递不同参数的时候），可以传入自定义的 Factory，而不是重写 ViewModelProvider 的某个函数。

看完 ViewModelProvider 的构造函数，再看返回 ViewModel 的函数：

接着看默认的 Factory 如何构造一个 ViewModel，以及自定义构造的话，要怎么做。

### NewInstanceFactory#getInstance()

```
// ViewModelProvider.java

    /**
     * Simple factory, which calls empty constructor on the give class.
     * 调用了给定类型的空构造函数
     */
    public static class NewInstanceFactory implements Factory {

        private static NewInstanceFactory sInstance;

        // 单例
        // 线程安全吗？ 不加 synchronized，所以不安全。
        // 那为何“敢这样写”？
        // 因为“几乎不会”遇到多线程的场景，该方法只会在
        // ViewModelProvider 构造函数中被调用，进而可知其“几乎只会”在 
        // Activity/Fragment 初始化时被调用，而它们只会在主线程被初始化，
        // 所以说“几乎不会”遇到多线程的场景；
        // 
        // 但仍然可以手动制造多线程的情况：
        // 在两个子线程持有 Activiy，并构造 ViewModelProvider 实例，
        // 线程不安全，会造成什么后果呢？
        // 指令重排序！
        // 造成 线程A 赋值了却没初始化，线程B 判空不为空，直接返回为初始化的对象
        @NonNull
        static NewInstanceFactory getInstance() {
            if (sInstance == null) {
                sInstance = new NewInstanceFactory();
            }
            return sInstance;
        }

        @SuppressWarnings("ClassNewInstance")
        @NonNull
        @Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            //noinspection TryWithIdenticalCatches
            try {
                return modelClass.newInstance();
            }...
        }
    }
```

调用了给定类型的空构造函数，很简单的实现。
如果想自定义 Factory，实现 create 函数的逻辑呢？
调用带 Factory 的该构造函数即可：

### Factory


```
// ViewModelProvider.java

    public interface Factory {
        /**
         * Creates a new instance of the given {@code Class}.
         * <p>
         *
         * @param modelClass a {@code Class} whose instance is requested
         * @param <T>        The type parameter for the ViewModel.
         * @return a newly created ViewModel
         */
        @NonNull
        <T extends ViewModel> T create(@NonNull Class<T> modelClass);
    }
    
    abstract static class KeyedFactory extends OnRequeryFactory implements Factory {
        /**
         * Creates a new instance of the given {@code Class}.
         *
         * @param key a key associated with the requested ViewModel
         * @param modelClass a {@code Class} whose instance is requested
         * @param <T>        The type parameter for the ViewModel.
         * @return a newly created ViewModel
         */
        @NonNull
        public abstract <T extends ViewModel> T create(@NonNull String key,
                @NonNull Class<T> modelClass);

        @NonNull
        @Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            throw new UnsupportedOperationException("create(String, Class<?>) must be called on "
                    + "implementaions of KeyedFactory");
        }
    }
    
    static class OnRequeryFactory {
        void onRequery(@NonNull ViewModel viewModel) {
        }
    }
```

上述的两个 Factory 接口的定义，特别注意第二个 KeyedFactory，继承了 OnRequeryFactory 类并实现了 Factory 接口，这有个有趣的地方，默认不允许调用不带 key 的 create 方法，会抛异常。

那么带 key 的 create 方法，什么时候调用，有什么特别的作用呢？

### SavedStateViewModelFactory

这个是 ComponentActivity、Fragment 默认带有的 Facroty。
ComponentActivity、Fragment 都实现了 HasDefaultViewModelProviderFactory 接口。

```
// ComponentActivity

    @NonNull
    @Override
    public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mDefaultFactory == null) {
            mDefaultFactory = new SavedStateViewModelFactory(
                    getApplication(),
                    this,
                    getIntent() != null ? getIntent().getExtras() : null);
        }
        return mDefaultFactory;
    }
```

所以在 Activity、Fragment 中 ViewModelProvider 用的 Factory 都是 SavedStateViewModelFactory，而不是上述的 NewInstanceFactory。

看看 SavedStateViewModelFactory 是怎么创建 ViewModel，即怎么调用 ViewModel 的构造函数，怎么给其构造函数传参数的。

```
// SavedStateViewModelFactory.java

public final class SavedStateViewModelFactory extends ViewModelProvider.KeyedFactory {

    ...

    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull String key, @NonNull Class<T> modelClass) {
    
        // 1，ViewModel 还有 AndroidViewModel 类型
        // 简答理解就是带有 Application 对象的 ViewModel
        boolean isAndroidViewModel = AndroidViewModel.class.isAssignableFrom(modelClass);
        
        // 2，查找匹配的构造函数
        Constructor<T> constructor;
        if (isAndroidViewModel) {
            constructor = findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE);
        } else {
            constructor = findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE);
        }
        
        // 3 不需要 SavedStateHandle 时，委托给 mFactory 创建 ViewModel
        // 这个 mFactory，是 ViewModelProvider.AndroidViewModelFactory
        // doesn't need SavedStateHandle
        if (constructor == null) {
            return mFactory.create(modelClass);
        }

        // 4，key 居然是跟创建 SavedStateHandle 有关
        SavedStateHandleController controller = SavedStateHandleController.create(
                mSavedStateRegistry, mLifecycle, key, mDefaultArgs);
                
        // 5，调用构造函数，把 SavedStateHandle 实例传递进去
        // 这也是为什么，在 ViewModel 中直接声明一个带 SavedStateHandle 参数的构造函数，不需要写 Factory 也能拿到 SavedStateHandle 实例的原因。
        try {
            T viewmodel;
            if (isAndroidViewModel) {
                viewmodel = constructor.newInstance(mApplication, controller.getHandle());
            } else {
                viewmodel = constructor.newInstance(controller.getHandle());
            }
            
            // 6，SavedStateHandleController 还会被作为 tag 设置给 ViewModel
            // 为何如此呢？
            viewmodel.setTagIfAbsent(TAG_SAVED_STATE_HANDLE_CONTROLLER, controller);
            return viewmodel;
        }...
    }
```

这里仅初步分析代码，没有深入分析。
从该段代码看，SavedStateViewModelFactory 创建 ViewModel，会帮忙创建一个 SavedStateHandle 对象，传递给其构造函数；如果需要创建的 ViewModel 类型是  AndroidViewModel 类型，则传递一个 Application 对象，看注释4，5；
这些事都是帮你做的。
所以在自己写代码时，如果想创建一个构造函数自动传入 Application 对象和 SavedStateHandle 对象的 ViewModel，直接定义一个带着两个类型参数的构造函数，并继承 
AndroidViewModel 即可。

如果不需要 SavedStateHandle 时，委托给 ViewModelProvider.AndroidViewModelFactory 创建 ViewModel。


```
// ViewModelProvider.AndroidViewModelFactory

        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
                //noinspection TryWithIdenticalCatches
                try {
                    // 1
                    return modelClass.getConstructor(Application.class).newInstance(mApplication);
                } ...
            }
            // 2
            return super.create(modelClass);
        }
```

逻辑是：
如果需要创建的 ViewModel 类型是 AndroidViewModel 类型，则传递一个 Application 对象，注释1的地方；

当需要创建的 ViewModel 类型既不是 AndroidViewModel 类型，也不需要 SavedStateHandle 对象，会走到注释2的地方，调用空构造函数。

### 自定义一个 Factory

反射如何调用带参数的构造函数？
参考该例子：

![](https://img-blog.csdnimg.cn/20190915184848112.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMzc3NzQ5,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20190915192036556.png)

### 中场小结

上述源码分析，都在于怎么实现一个 Factory，怎么创建一个 ViewModel，默认的 Factory 是怎么做的。到此可以回答这些问题了。

实现一个 Factory 就是实现其 create 接口，根据如此类型，反射调用构造函数即可。
默认的 Factory，在 Activity/Fragment 的是 SavedStateViewModelFactory，
会帮你创建 Application 对象和 SavedStateHandle 对象，当然你得在自定义的 ViewModel 的构造函数中声明两个类型的参数；如果不写构造参数，也会调用空构造函数，不过委托了 ViewModelProvider.AndroidViewModelFactory。


### get(modelClass)

```
// ViewModelProvider.java 

    /**
     * Returns an existing ViewModel or creates a new one in the scope (usually, a fragment or
     * an activity), associated with this ViewModelProvider.
     * <p>
     * The created ViewModel is associated with the given scope and will be retained
     * as long as the scope is alive (e.g. if it is an activity, until it is
     * finished or process is killed).
     */
    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
    
    private static final String DEFAULT_KEY =
            "androidx.lifecycle.ViewModelProvider.DefaultKey";
```

注释解析的也清楚，返回一个已存在的 ViewModel；
或者创建一个新的 ViewModel，放入 scope（作用域，通常是 Activit 或 Fragment），关联当前的 ViewModelProvider。

ViewModel 存活时间会跟 scope 一样长。

### get(key, modelClass)

```
// ViewModelProvider.java

    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);
        // 1，从 mViewModelStore 中取 ViewModel
        // 这里取到的 viewModel，可能是空，也可能是不同与 modelClass 类型
        // isInstance 方法，即判空，也判断同类型
        if (modelClass.isInstance(viewModel)) {
        
            // onRequery 函数原本是空实现，让子类拓展逻辑的地方
            // 在子类查找到已存在 ViewModel 时的一个回调
            // 当旋转时，Activity 销毁有重建，再获取 ViewModel 时，
            // 会出现 ViewModel 已存在的情况，会走到这里
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;//2，返回已存在的
        } 
        ...
        
        //3，新创建一个，通过 Factory
        //
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
        } else {
            viewModel = (mFactory).create(modelClass);
        }
        mViewModelStore.put(key, viewModel);//4，保存
        return (T) viewModel;//5，返回新创建的
    }
```

注释1，可知 **ViewModel 存储于 mViewModelStore 之中**。
一个 ViewModelProvider 持有一个 mViewModelStore，
mViewModelStore 是构造函数中传入的 ViewModelStoreOwner 中获取的，
也就是从 Activity、Fragment 中拿到的；

注释3，可知会根据 Factory 的类型调用参数不同的方法来创建 ViewModel 对象，
分别为带 key 和 不带 key 两种 Factory，
KeyedFactory、Factory 都是定义在 ViewModelProvider 的两个接口。

注释1和注释5，可知存取都在 ViewModelStore，接着看看 ViewModelStore 是怎么实现的。

### ViewModelStore

```
// ViewModelStore.java

/**
 * Class to store ViewModels.
 * 
 * An instance of ViewModelStore must be retained through configuration changes:
 * if an owner of this ViewModelStore is destroyed and recreated 
 * due to configuration changes, new instance of an owner should 
 * still have the same old instance of ViewModelStore.
 * 
 * If an owner of this ViewModelStore is destroyed 
 * and is not going to be recreated,
 * then it should call #clear() on this ViewModelStore, so ViewModels would
 * be notified that they are no longer used.
 * 
 * Use ViewModelStoreOwner#getViewModelStore() to 
 * retrieve a ViewModelStore for activities and fragments.
 */
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        // 返回的是被替换的旧值，因而会清空操作
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

可知 ViewModelStore 基于 HashMap 做存储。

其余函数注释写的很清楚：
存储 ViewModel ；
owner 的 configuration 改变后，owner 的新实例，拿到的还是原来的 store；
clear 方法，在 Activity、Fragment 等 owner 销毁后调用到。
使用 ViewModelStoreOwner#getViewModelStore() 可获得一个 ViewModelStore。

### 思考

ViewModelStore 基于 HashMap 做存储，为何不用 ArrayMap？
存的 ViewModel 数量不大，这里做优化，意义不大，节省不了多少内存。

问题：多个 Activity 会持有同一个 ViewModelStore？
答：不会！一个 Activity 一个 ViewModelStore。
一个 Activity 可持有多个 ViewModelProvider，
一个 ViewModelProvider 就可以创建多个 ViewModel，
这些 ViewModel 都存储于同一个 Activity/Fragment 中，
一个 Activity/Fragment 持有一个 ViewModelStore 中。
也就是说，一个 Activity/Fragment一般持有一个或几个 ViewModel，超过10个的都不常见。
因此，此处没必要为了节省内存而使用 ArrayMap。

问：谁实现 ViewModelStoreOwner 接口，持有 ViewModelStore 引用？
答：ComponentActivity、Fragment 的 FragmentManager 中。
ComponentActivity 的继承关系：
```
AppCompatActivity 
ext FragmentActivity 
ext ComponentActivity
```

### ViewModelStoreOwner

接口：

```
public interface ViewModelStoreOwner {
    /**
     * Returns owned {@link ViewModelStore}
     *
     * @return a {@code ViewModelStore}
     */
    @NonNull
    ViewModelStore getViewModelStore();
}
```

实现类：

```
// AppCompatActivity

    // Lazily recreated from NonConfigurationInstances by getViewModelStore()
    private ViewModelStore mViewModelStore;
    private ViewModelProvider.Factory mDefaultFactory;
    
    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
```


```
// Fragment.java

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (mFragmentManager == null) {
            throw new IllegalStateException("Can't access ViewModels from detached fragment");
        }
        return mFragmentManager.getViewModelStore(this);
    }
```

### ViewModel

ViewModel 的源码非常简单，主要有
- 空实现的 onCleared() 方法，让子类重写
- final 的 clear() 方法，清空存储的 tag
- setTagIfAbsent/getTag 方法对
    - 用 map 存储一个任意类型的 tag，ket 必须是 String
    - 若 tag 是 Closeable 类型，则执行 clear() 方法时会关闭该 tag

## 小结

ViewModel 到此初步完整的分析完。

到此先想想，ViewModel 的基本作用是什么？

> 辅助 Activity/Fragment，保存 View 的状态数据，不因旋转（配置更改）而销毁。
> 但 Activity/Fragment 销毁而不重建，ViewModel 也应该销毁。
> Fragment 之间可以共享宿主 Activity 的 ViewModel。
> 一个 Activity 可以有多个 ViewModel，
> 一个 ViewModel 只能属于一个 Activity/Fragment；
> 等等。

因此，需要怎么设计 ViewModel 的架构？

先看一张图：
![屏幕快照 2021-05-17 下午3.42.08](/img/屏幕快照 2021-05-17 下午3.42.08.png)


自己的总结：

参考：[重学安卓：有了 Jetpack ViewModel . . . 真的可以为所欲为！](https://xiaozhuanlan.com/topic/6257931840)

> 
> Google 是这么设计的：
> 
> Activity 和 Fragment，是作为 ViewModelStoreOwner 的实现，每个 ViewModelStoreOwner 持有着唯一的一个 ViewModelStore，也即 每个视图控制器都持有自己的一个 ViewModelStore。
> 
> **ViewModelStore 的作用是维护一个 Map，来管理「视图控制器」所持有的所有 ViewModel。**
> 
> 所以三者的关系是，ViewModelStoreOwner 持有 ViewModelStore 持有 ViewModel。
> 
> 并且 当作为 ViewModelStoreOwner 的视图控制器被 destory 时（重建的情况除外），ViewModelStore 会被通知去完成清除状态操作，从而将 Map 中管理着的 ViewModel 全部走一遍 clear 方法，并且清空 Map。
> 
> （clear 方法是 final 级，不可修改，clear 方法中包含 onClear 钩子，开发者可重写 onClear 方法来自定义数据的清空）

<br>

> 那么，在有了上述 ViewModelStoreOwner - ViewModelStore - Map - ViewModel 设计的 兜底 后，如何做到 “状态共享以及作用域可控” 呢？

> 很简单 —— **通过 “工厂模式” 来搞定**：

> 在 Fragment 页面中创建 ViewModelProvider 实例，并在入参中注入 ViewModelStoreOwner，参照上述的兜底设计，我们可知，此处即 通过 ViewModelStoreOwner 来反映作用域：

> 譬如上述新建的 ViewModelProvider 传入的 ViewModelStoreOwner 的是 getActivity() ，那么它实际上是透过 “该 Activity 持有的 ViewModelStore 持有的 Map” 来 put 或 get ViewModel —— 如果 Map 中已包含目标 ViewModel 实例，那么当页面获取到该 ViewModel 实例，便达成了与该 Activity 或隶属于该 Activity 的其他 Fragment 共享 ViewModel 实例的目的。

> 同理，如果传入的 ViewModelStoreOwner 的是 Fragment.this，那么它的作用域仅限于本 Fragment，并且当 Fragment 离场时，该 ViewModel 实例也会连同 Fragment 一起被销毁。

> 换言之，同一个 ViewModel 子类，基于不同的作用域，获取到的实例并非同一个。根据传入的 ViewModelStoreOwner 我们便能拿到符合实际场景所需的 ViewModel 实例。

抱歉，在读完这一段，我没有提炼出“工厂模式”类实现 “状态共享以及作用域可控”的效果的论据。
我认为：
通过独立的 ViewModelStoreOwner，存储跟自己生命周期一致的 ViewModel；
实现 Fragment 和 Fragment 都有跟自己同生命周期的 ViewModel，**实现“作用域可控”的效果**；
在配合 Activity 跟自己名下的 Fragment 的关系，Fragment 可以获取宿主 Activity 的引用，使得同宿主的 Fragment 可以获取到同一个 ViewModel，**实现“状态共享”的效果**。