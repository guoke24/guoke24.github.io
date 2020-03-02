---
layout:     post
title:      "Handler 如何绑定 Thread 线程"
subtitle:   ""
date:       2020-01-30 12:00:00
author:     "GuoHao"
header-img: "img/waittouse.jpg"
catalog: true
tags:
    - Android
    - 源码
    - Handler
    - ThreadLocal
---

# Handler 的构造函数
Handler 在被实例化时，其构造函数，会绑定当前线程的 Looper 实例。<br>
其源码如下：

```
// Handler 类

    public Handler() {
        this(null, false);
    }
```

```
// Handler 类

    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

	// 1-1 绑定当前线程的 Looper 实例，并获得引用
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
1-1处，调用了 Looper.myLooper() 来获取一个 looper 实例。<br>
继续 Looper.myLooper() 源码：

# Looper.myLooper() 

```
// Looper 类

public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

# Looper 的成员变量 sThreadLocal
```
// Looper 类

    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
由此可知，静态函数 myLooper() 内是委托静态成员变量 sThreadLocal 的 get 函数来取 Looper 实例的。<br>

sThreadLocal 是一个静态成员变量，意味着该变量是进程唯一的。
这样使得进程内的所有线程，都会委托该变量来统一进行 Looper 绑定。<br>

sThreadLocal 属于 ThreadLocal 类，再看其 get 函数内部： 
```
// ThreadLocal 类

    public T get() {
        Thread t = Thread.currentThread();
        // 2-1 拿到 本地线程map
        ThreadLocalMap map = getMap(t); 
        if (map != null) {
            // 2-2 以自身为 key，取出 value
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;// 2-3
                return result;
            }
        }
        return setInitialValue();
    }
```

继续看 2-1 处，

```
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
getMap 函数拿到的 map 是当前线程 Thread 类实例的成员变量 threadLocals。
通过变量 map 存取的数据，都是线程特有的。

# ThreadLocal 的静态内部类 ThreadLocalMap
```
// Thread 类

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
回看2-1 处，getMap 函数，拿到的引用是当前线程的成员变量 threadLocals，从类型名字看叫作「线程本地 map」，即 ThreadLocal 的 内部静态类 ThreadLocalMap。从名字看，这个类型的作用就是帮线程存储一些「本地数据的 map」。<br>

回到 2-2 处，再看 ThreadLocalMap 的 getEntry 函数：
```
// ThreadLocal.ThreadLocalMap 类

        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```
从 ThreadLocalMap 内部变量 table 中取出  Entry 类实例 e 返回，在注释2-3处，取 e.value 得到 Looper 实例。而 Entry 是一个弱引用类型：WeakReference< ThreadLocal< ? > >，其泛型 ThreadLocal< ? > 作为 key，其 value 就是 < ? > 中指定的类型实例。再回顾这段代码：
```
// Looper 类

    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

到此，可知，new Handler 的时候，会委托 Looper 类的静态函数 myLooper() ，再委托其的静态成员 sThreadLocal 的 get 函数，从当前的线程实例的成员变量 threadLocals 中取出 key 值为 ThreadLocal< Looper >的 value 值，即一个 Looper 对象。<br>

关系链：
Thread 对象 -- Looper 类 -- sThreadLocal 实例 -- 当前 Thread 实例.threadLocals<br>

那么接下来的问题，什么时候往当前 Thread 实例的成员变量 threadLocals 中放入一个 Looper 实例呢？<br>

那就是在当前线程执行 Looper.prepare() 的时候。

# Looper.prepare()

```
// Looper 类

    /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed)); //3-1
    }
```
set 前有一个判断，使得只能放入一个 Looper 实例到当前线程。

接着 3-1 ，
```
// ThreadLocal 类

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value); // 4-1
        else
            createMap(t, value); // 4-2
    }


//  。。。

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
在 ThreadLocal 的 set 函数内，先拿到当前的线程实例的成员变量 threadLocals ，若为空就去创建一个。
其 ThreadLocalMap 类的构造函数为：

```
// ThreadLocal.ThreadLocalMap 类

        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```
创建的同时就放一个 Object 类型的  firstValue  到 table 数组变量。
在 Looper 这里，放入的键值对就是 ThreadLocal<Looper> -- new Looper() 。 

再看 4-1，
```
// ThreadLocal.ThreadLocalMap 类

        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
最终，ThreadLocal<Looper> -- new Looper()  这个键值对，被保存在 ThreadLocal.ThreadLocalMap 类实例的 table 数组变量中。
 
# 总结
到此可知，已经可以回答文章标题的问题了，Handler 如何绑定 Thread 线程？<br>
当前线程在 new Handler 之前，先委托了 Looper 的静态函数 prapare()，new 了一个 Looper 实例，再委托其 静态成员变量 sThreadLocal 的 set 函数，把刚 new 的 Looper 实例做保存操作，并且在保存之前，先通过 sThreadLocal 的 get 函数，判断是否已经保存有 Looper 实例，如果有就报错。这样就能保证一个线程只保存一个 Looper 实例。<br>

这样的机制，使得在同一个线程内，无论 new 多少个 Handler，通过 Looper 类的静态函数 myLooper() 拿到的 Looper 实例都是同一个。也就是说，同一个线程内的 Handler，默认都是绑定当前线程的，且绑定同一个 Looper 实例。<br>

关系链：
Thread 实例 -- Looper 类 -- Looper 类.sThreadLocal -- Thread 实例.threadLocals <br>

| Thread 实例  | Looper 类  | sThreadLocal 实例（ThreadLocal 类） | Thread 实例.threadLocals（ThreadLocal. ThreadLocalMap 类）  |
|:----------|:----------|:----------|:----------|
| 调用 Looper#prepare 函数   | sThreadLocal.get() 判断当前线程是否已有Looper；new 一个 Looper实例，委托 sThreadLocal.set 函数进行保存    |  通过该调用 Thread.currentThread()，使得后续操作都是基于当前线程；取得当前本地线程map，通过其 set 和 getEntry 函数来存取 Looper 实例    | 内部维护一个 table 数组，其元素是弱引用类型的子类 Entry，以 ThreadLocal<?> 为 key 值，Object 类型为 value值；提供 set 和 getEntry 函数给外部进行存取操作  |
|    |     |  |    |

# 延伸
更详细的内容，可以参考文章 [[ ThreadLocal 原理 ]] 。<br>

Looper.prepare()  函数，在主线程中，已经在 ActivityThread 的 main 函数内执行过一次，所以不需要开发者手动调用，但如果在子线程中 new Handler，就需要先执行一次 Looper.prepare()  函数。

