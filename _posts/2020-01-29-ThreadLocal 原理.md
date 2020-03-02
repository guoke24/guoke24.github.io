---
layout:     post
title:      "ThreadLocal 原理"
subtitle:   ""
date:       2020-01-29 12:00:00
author:     "GuoHao"
header-img: "img/waittouse.jpg"
catalog: true
tags:
    - Android
    - 源码
    - Handler
    - ThreadLocal
---
# 总览

ThreadLocal 类的声明，带一个泛型 T：

```
public class ThreadLocal<T>{

}
```

ThreadLocal 类的有一个静态内部类 ThreadLocal.ThreadLocalMap，声明：
```
//  ThreadLocal 类

static class ThreadLocalMap {
```

ThreadLocal 类，开放给外部的 set 函数：
```
//  ThreadLocal 类

    public void set(T value) {
	// 先获得当前执行的线程
        Thread t = Thread.currentThread();
	// 获得线程对象内部的成员变量，ThreadLocalMap 类的实例 threadLocals
        ThreadLocalMap map = getMap(t); // 3-1
        if (map != null)
	     // 直接存 this<T> - value 到 map
            map.set(this, value); // 3-2
        else
	    // 创建后再存
            createMap(t, value);// 3-3
    }
```

注释3-1，getMap(t)，得到当前线程 Thread 实例的成员变量 threadLocals
```
//  ThreadLocal 类

    // 返回 ThreadLocalMap 类型的实例 threadLocals
    // 且该实例是当前线程 Thread 实例的成员变量
    // ThreadLocalMap，看名字，线程本地 Map，就是帮助线程存储一些本地的数据，以键值对的形式
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
t.threadLocals，是 ThreadLocal 的静态内部类 ThreadLocalMap

```
// Thread 类

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

注释3-3，实例化 threadLocals
```
//  ThreadLocal 类

    // Thread 的成员变量 threadLocals ，在此处实例化
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

总的来说，可以简单的小结：ThreadLocal 类的作用，就是帮助一个线程对象，存一个「任意类型的变量」到「线程本地 map」中。<br>

# key 值重复问题
引申出一个问题，3-2 处，存键值对的时候，key 值都用自己，会不会出现 key 值重复的问题？<br>

答：设计如此，ThreadLocal 类中「key 值都用自己」这样的设计，就是想做到唯一性 ：

```
一个 ThreadLocal 类实例，只能帮助每个线程实例存一个 泛型实例，不能多存，这样的唯一性。
```
举例：
T1 和 T2 两个线程，持有相同的一个 ThreadLocal 类实例，多次操作存不同的变量，观察结果，会不会出现线程间混淆？同线程内是否新值会替代旧值？<br>
第一个问题答案是不会混淆，因为不同线程，通过同一个 ThreadLocal 实例，存到各自的成员变量 threadLocals 中去了，不会混淆；<br>
第二个答案是，新值会代替旧值。

# ThreadLocal.ThreadLocalMap 类的实现

```
    static class ThreadLocalMap {

	// 弱引用，泛型为 <ThreadLocal<?>>
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0

        /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        /**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }

	// 。。。

        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

	// 。。。

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
ThreadLocal.ThreadLocalMap 类中，持有一个成员变量 table，是一个弱引用类型的数组，并且指定了泛型为 ThreadLocal<?>，其 set 和 get 函数都是往 table 里存取 Object 类型的数据，也就是可以存任意类型的数据。<br>
set 函数的第一个参数为 ThreadLocal<?> key，也就是说，往这里存数据，必须有一个 ThreadLocal<?> 的实例，并将其作为 key 值传进来。而刚好，在 ThreadLocal 类的 set 函数中，正是将自身 this 传给了 ThreadLocal.ThreadLocalMap 实例的 set 函数。
```
//  ThreadLocal 类

    public void set(T value) {
	// 先获得当前执行的线程
        Thread t = Thread.currentThread();
	// 获得线程对象内部的成员变量，ThreadLocalMap 类的对象 threadLocals
        ThreadLocalMap map = getMap(t); // 3-1
        if (map != null)
	     // 直接存 this<T> - value 到 map
            map.set(this, value); // 3-2
        else
	    // 创建后再存
            createMap(t, value);// 3-3
    }
```


# Looper 案例
每一个线程，都对应一个 Thread 对象，一个 Thread 对象，如果希望 Handler 的 handleMessage 函数在自身线程执行，就需要委托 Looper 类，new 一个 Looper，
再委托 Looper 类持有的静态成员、进程唯一的 sThreadLocal 实例，将 sThreadLocal 实例 和 new Looper  组成一个键值对，保存于该 Thread 实例的 线程本地map，即成员变量 threadLocals 中。<br>

关系链：
Thread 实例 -- Looper 类 -- Looper 类.sThreadLocal -- Thread 实例.threadLocals <br>

| Thread 实例  | Looper 类  | sThreadLocal 实例（ThreadLocal 类） | Thread 实例.threadLocals（ThreadLocal. ThreadLocalMap 类）  |
|:----------|:----------|:----------|:----------|
| 调用 Looper#prepare 函数   | sThreadLocal.get() 判断当前线程是否已有Looper；new 一个 Looper实例，委托 sThreadLocal.set 函数进行保存    |  通过该调用 Thread.currentThread()，使得后续操作都是基于当前线程；取得当前本地线程map，通过其 set 和 getEntry 函数来存取 Looper 实例    | 内部维护一个 table 数组，其元素是弱引用类型的子类 Entry，以 ThreadLocal<?> 为 key 值，Object 类型为 value值；提供 set 和 getEntry 函数给外部进行存取操作  |
|    |     |  |    |



每当 Thread 实例直接或间接地委托 sThreadLocal 实例 存取 Looper 对象的时候，sThreadLocal 实例调用 Thread.currentThread() 得到当前 Thread 对象，进而得到 Thread 对象的成员变量 threadLocals，再以 threadLocals 为容器进行存储操作。
这样的机制，使得 Thread 对象跟 Looper 对象产生一一对应的关系，因为每个 Thread 对象所对应的 Looper 对象都是存放在自身的成员变量 threadLocals 中，不会跟别的线程的 Looper 产生混淆。<br>

上述的 Thread 对象的成员变量 threadLocals 的类型 ThreadLocal.ThreadLocalMap，是  sThreadLocal 实例类型 ThreadLocal 的静态内部类。