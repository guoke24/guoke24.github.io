---
layout:     post  
title:      "ThreadLoacl 原理-源码梳理"  
subtitle:   "每个线程的私人助理"  
date:       2020-03-17 12:00:00  
author:     "GuoHao"  
header-img: "img/waittouse.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - Handler  
    - ThreadLocal

---

### 参考

大神级的细节，表述和逻辑：[ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html)

### 总览

引用参考文章的一段话： <br>

> 先回答两个问题：

> 1，什么是ThreadLocal？<br>
> <br>
ThreadLocal 类顾名思义可以理解为线程本地变量。也就是说如果定义了一个ThreadLocal，每个线程往这个ThreadLocal中读写是线程隔离，互相之间不会影响的。它提供了一种将可变数据通过每个线程有自己的独立副本从而实现线程封闭的机制。

> 2，它大致的实现思路是怎样的？<br>
> <br>
Thread 类有一个类型为 ThreadLocal.ThreadLocalMap 的实例变量threadLocals，也就是说每个线程有一个自己的 ThreadLocalMap。ThreadLocalMap 有自己的独立实现，可以简单地将它的 key 视作 ThreadLocal，value 为代码中放入的值（实际上key并不是ThreadLocal本身，而是它的一个弱引用）。每个线程在往某个 ThreadLocal 里塞值的时候，都会往自己的ThreadLocalMap 里存，读也是以某个 ThreadLocal 作为引用，在自己的map里找对应的 key，从而实现了线程隔离。<br>

<br>

### 存储结构 -- 弱引用的键值

在此先引用参考文章的文字：<br>

> ThreadLocalMap 是个 map（注意不要与java.util.map混为一谈，这里指的是概念上的map），当然得要有自己的 key 和 value，上面回答的问题2中也已经提及，我们可以将其简单视作 key 为 ThreadLocal，value 为实际放入的值。之所以说是简单视作，因为实际上 ThreadLocalMap 中存放的是 ThreadLocal 的弱引用。我们来看看 ThreadLocalMap 里的节点是如何定义的。
> 
> Entry 便是 ThreadLocalMap 里定义的节点，它继承了 WeakReference 类，定义了一个类型为 Object 的 value，用于存放塞到 ThreadLocal 里的值。

> 4.2 为什么要弱引用

> 读到这里，如果不问不答为什么是这样的定义形式，为什么要用弱引用，等于没读懂源码。
>  
因为如果这里使用普通的 key-value 形式来定义存储结构，实质上就会造成节点的生命周期与线程强绑定，只要线程没有销毁，那么节点在 GC 分析中一直处于可达状态，没办法被回收，而程序本身也无法判断是否可以清理节点。弱引用是 Java 中四档引用的第三档，比软引用更加弱一些，如果一个对象没有强引用链可达，那么一般活不过下一次 GC。当某个 ThreadLocal 已经没有强引用可达，则随着它被垃圾回收，在ThreadLocalMap 里对应的 Entry 的键值会失效，这为 ThreadLocalMap 本身的垃圾清理提供了便利。

<br>


**用我自己的话再概括一遍：**<br>

在 ThreadLocal.ThreadLocalMap 的内部维护着一个 table 数组，<br>
用来存储 ThreadLocal 塞进来的 Object 类型的值，table 数组的元素是 Entry 类。<br>

Entry 是弱引用类 WeakReference 的派生类，其引用对象是 ThreadLocal<?> 类的实例。<br>
之所以使用弱引用是因为方便垃圾回收。

相关源码如下：

```
    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        
        ...

        private Entry[] table;
```
<br>


### 存储的槽位 -- 数组下标

#### 魔数累加，(2的幂)取模

根据上述参考的文章，可知 ThreadLocal 为每个被创建的实例，以累加魔数 `0x61c88647` 的方式生成一个 hashcode（threadLocalHashCode）作为其 ID，便于这样一种情况：  
同一个线程下，多个 ThreadLocal 实例拿的是同一个 ThreadLocalMap ，往其 table 数组_存取_数据时，可以根据这多个 ThreadLocal 实例的不同 hashcode，**通过跟「2的幂」做取模运算**，计算出分布很均匀的 table 数组下标（也称为：**槽位，slot**）。<br>
<br>
这里补充一点：ThreadLocalMap 的 table 的下标是环形移动的，初始容量固定为 16。

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
        ......
    
        /**
         * 必须为 2 的幂，a power of two
         */
        private static final int INITIAL_CAPACITY = 16;

        private Entry[] table;

        /**
         * Increment i modulo len. 环形意义的下一个索引
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        /**
         * Decrement i modulo len. 环形意义的上一个索引
         */
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
        
    
        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            // 通过 threadLocalHashCode 计算出 table 的下标 i
            int i = key.threadLocalHashCode & (len-1);
            ......
            tab[i] = new Entry(key, value);
        ......
```
<br>
为了更形象的理解这个计算过程，特意写了一个 demo 来验证一下...<br>
<br>
先看 ThreadLocal 源码：<br>

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

public class ThreadLocal<T> {
   
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```
<br>
可以看到，<br>
静态的原子计数器 nextHashCode，<br>
静态常量魔数 HASH_INCREMENT，<br>
静态方法 nextHashCode，<br>
因为静态的都是类特有的，<br>

所以，在每个 ThreadLocal 实例创建时，得到的 threadLocalHashCode 值，<br>
都是用上一个实例的 threadLocalHashCode 加上魔数 HASH_INCREMENT 得到的。<br>
因为 AtomicInteger 初始化的值为 0，所以第一个实例的 threadLocalHashCode 也为 0。<br>

#### 验证

下面通过 demo 代码验证一下：<br>

```
    private static final int HASH_INCREMENT = 0x61c88647;

    private static AtomicInteger nextHashCode =
            new AtomicInteger();

    // 测试 threadLocalHashCode 的产生
    static void testThreadLocalHashCode(){
    
        // 模拟第1个 ThreadLocal<T> thrdLcl1 = new ThreadLocal<T>();// threadLocalHashCode1
        int threadLocalHashCode1 = nextHashCode();

        // 模拟第2个 ThreadLocal<T> thrdLcl2 = new ThreadLocal<T>();// threadLocalHashCode2
        int threadLocalHashCode2 = nextHashCode();

        // 模拟第3到第20个
        int threadLocalHashCode3 = nextHashCode();
        int threadLocalHashCode4 = nextHashCode();
        int threadLocalHashCode5 = nextHashCode();
        int threadLocalHashCode6 = nextHashCode();
        int threadLocalHashCode7 = nextHashCode();
        int threadLocalHashCode8 = nextHashCode();
        int threadLocalHashCode9 = nextHashCode();
        int threadLocalHashCode10 = nextHashCode();
        int threadLocalHashCode11 = nextHashCode();
        int threadLocalHashCode12 = nextHashCode();
        int threadLocalHashCode13 = nextHashCode();
        int threadLocalHashCode14 = nextHashCode();
        int threadLocalHashCode15 = nextHashCode();
        int threadLocalHashCode16 = nextHashCode();
        int threadLocalHashCode17 = nextHashCode();
        int threadLocalHashCode18 = nextHashCode();
        int threadLocalHashCode19 = nextHashCode();
        int threadLocalHashCode20 = nextHashCode();


        // 同一个线程，可以新建多个 ThreadLocal<T> 实例，但操作的 ThreadLocalMap 对象都是同一个
        // 同一个线程的多个 ThreadLocal<T> 实例，threadLocalHashCode 值不一样
        // 这个值，会影响其在 ThreadLocalMap 中保存数据时用到的 table 数组的下标。
        // 具体计算方式就是
        // 对于2的幂作为模数取模，可以用&(2^n-1)来替代%2^n，位运算比取模效率高很多
        int i1 = threadLocalHashCode1 & (16-1);
        int i2 = threadLocalHashCode2 & (16-1);
        int i3 = threadLocalHashCode3 & (16-1);
        int i4 = threadLocalHashCode4 & (16-1);
        int i5 = threadLocalHashCode5 & (16-1);
        int i6 = threadLocalHashCode6 & (16-1);
        int i7 = threadLocalHashCode7 & (16-1);
        int i8 = threadLocalHashCode8 & (16-1);
        int i9 = threadLocalHashCode9 & (16-1);
        int i10 = threadLocalHashCode10 & (16-1);
        int i11 = threadLocalHashCode11 & (16-1);
        int i12 = threadLocalHashCode12 & (16-1);
        int i13 = threadLocalHashCode13 & (16-1);
        int i14 = threadLocalHashCode14 & (16-1);
        int i15 = threadLocalHashCode15 & (16-1);
        int i16 = threadLocalHashCode16 & (16-1);
        int i17 = threadLocalHashCode17 & (16-1);
        int i18 = threadLocalHashCode18 & (16-1);
        int i19 = threadLocalHashCode19 & (16-1);
        int i20 = threadLocalHashCode20 & (16-1);

        System.out.println("threadLocalHashCode1 = " + threadLocalHashCode1);
        System.out.println("threadLocalHashCode2 = " + threadLocalHashCode2);
        System.out.println("threadLocalHashCode3 = " + threadLocalHashCode3);
        System.out.println("......");

        System.out.println("i1 = " + i1);
        System.out.println("i2 = " + i2);
        System.out.println("i3 = " + i3);
        System.out.println("i4 = " + i4);
        System.out.println("i5 = " + i5);
        System.out.println("i6 = " + i6);
        System.out.println("i7 = " + i7);
        System.out.println("i8 = " + i8);
        System.out.println("i9 = " + i9);
        System.out.println("i10 = " + i10);
        System.out.println("i11 = " + i11);
        System.out.println("i12 = " + i12);
        System.out.println("i13 = " + i13);
        System.out.println("i14 = " + i14);
        System.out.println("i15 = " + i15);
        System.out.println("i16 = " + i16);
        System.out.println("i17 = " + i17);
        System.out.println("i18 = " + i18);
        System.out.println("i19 = " + i19);
        System.out.println("i20 = " + i20);

    }
```

<br>
输出结果：

>threadLocalHashCode1 = 0  
threadLocalHashCode2 = 1640531527  
threadLocalHashCode3 = -1013904242  
......  
i1 = 0 <br> 
i2 = 7 <br> 
i3 = 14 <br> 
i4 = 5 <br> 
i5 = 12 <br> 
i6 = 3 <br> 
i7 = 10 <br> 
i8 = 1 <br> 
i9 = 8 <br>
i10 = 15 <br>
i11 = 6 <br>
i12 = 13 <br>
i13 = 4 <br>
i14 = 11 <br>
i15 = 2 <br>
i16 = 9 <br>
i17 = 0 <br>
i18 = 7 <br>
i19 = 14 <br>
i20 = 5 <br>

<br>
可以看到，从第 1 到 16 个 threadLocalHashCode 算出的下标（即槽位）都没有重复，且非常均匀； <br>
直到第 17 个 threadLocalHashCode 开始，其下标又开始跟 第 1 个一样... <br>
<br>
引用参考文章的一段话： <br>
> ThreadLocal类中有一个被final修饰的类型为int的threadLocalHashCode，它在该ThreadLocal被构造的时候就会生成，相当于一个ThreadLocal的ID
> 可以看出，它是在上一个被构造出的ThreadLocal的ID/threadLocalHashCode的基础上加上一个魔数0x61c88647的。<br>
> 
> 这个魔数的选取与斐波那契散列有关，0x61c88647对应的十进制为1640531527。斐波那契散列的乘数可以用(long) ((1L << 31) * (Math.sqrt(5) - 1))可以得到2654435769，如果把这个值给转为带符号的int，则会得到-1640531527。换句话说
(1L << 32) - (long) ((1L << 31) * (Math.sqrt(5) - 1))得到的结果就是1640531527也就是0x61c88647。<br>

> 通过理论与实践，当我们用0x61c88647作为魔数累加为每个ThreadLocal分配各自的ID也就是threadLocalHashCode再与2的幂取模，得到的结果分布很均匀。
ThreadLocalMap使用的是线性探测法，均匀分布的好处在于很快就能探测到下一个临近的可用slot，从而保证效率。这就回答了上文抛出的为什么大小要为2的幂的问题。为了优化效率。

> 对于& (INITIAL_CAPACITY - 1)，相信有过算法竞赛经验或是阅读源码较多的程序员，一看就明白，对于2的幂作为模数取模，可以用 “&(2^n - 1)” 来替代 “%2^n ”，位运算比取模效率高很多。至于为什么，因为对 2^n 取模，只要不是低n位对结果的贡献显然都是0，会影响结果的只能是低n位。

> 可以说在ThreadLocalMap中，形如key.threadLocalHashCode & (table.length - 1)（其中key为一个ThreadLocal实例）这样的代码片段实质上就是在求一个ThreadLocal实例的哈希值，只是在源码实现中没有将其抽为一个公用函数。

<br>
到此，明白了 ThreadLocal 往 ThreadLocalMap 塞数据的时候，是如何确定槽位（下标）的。

### 存取的具体实现

#### 存的实现

##### 流程图

直接来 set 函数的流程图：

流程图是截图的，不太清晰；清晰版的可参考csdn的同篇文章：[ThreadLoacl 原理](https://blog.csdn.net/gh1026385964/article/details/106450514)

![](/img/threadlocalmap_set函数流程图.png)

<br>
到此可知，set 函数有 6、8、13、14 这4个 return 点；<br>
其中，需要重点关注的有一下几点：<br>
第 5 点，调用 cleanSomeSlots() 函数；<br>
第 8 点，调用 rehash() 函数；<br>
第 14 点的替换陈旧的槽位操作，会调用 replaceStaleEntry(...) 函数，<br>
其内部还会调用 expungeStaleEntry(int i) 和 cleanSomeSlots() 函数进行清理。<br>


先看 replaceStaleEntry(...) 函数。<br>
接着再看 expungeStaleEntry(int i) 和 cleanSomeSlots() 函数的实现。<br>

分析源码前，先约定几个用词：

```
一个数组的每个元素，称形象的称为槽；
每个元素的下标，称为槽位；
元素引用的对象，称为 entry，也叫节点；
对于 ThreadLocalMap 中的 table 数组，其元素引用的对象 entry 包含一个 key-value 对；
但这个 key 是个弱引用，当被 GC 回收的时候，entry 的 key 就为空，在此称该元素为失效槽。
```
<br>

##### set(...) 函数

进入 set(...) 函数源码：<br>

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
        .......

        private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);// hash计算出下标

            // 向后线性探测
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) { // 环形意义的下一个
                ThreadLocal<?> k = e.get();

                if (k == key) { // 存在同 key 值的槽，替换 value
                    e.value = value;
                    return;
                }

                if (k == null) { // 存在失效槽，替换并清理
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            
            // 能来到这里，表示扫描到 tab[i] == null，才会跳出上面的循环；
            // 说明即没有同 key 槽，也没有失效槽（key被回收）；
            // 放入扫描到的新的空槽
            tab[i] = new Entry(key, value);
            int sz = ++size;
            
            // 从槽位i 往后尝试清理
            // 若没有清理 && 存储个数 >= 阈值
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash(); // 全量清理并扩容
        }
        
        .......
```

<br>

##### replaceStaleEntry(...) 函数

进入 replaceStaleEntry(...) 函数源码：<br>

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
    
        ......
        
        /**
         * 把 key-value 值放入 staleSlot 指向的槽位，
         * 并且清理 staleSlot 之外的失效槽位（key被回收的槽位）
         */ 
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {     
            // 调用函数进来这里，说明入参 staleSlot 的槽位是失效槽，即非空但key被回收。
            // 后续分析，把入参 staleSlot 的槽位简称入参槽位
            // 把没有 entry 的数组元素称为空槽
            
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // 暂定待清理槽位为 入参槽位
            int slotToExpunge = staleSlot;
            
            // 向前线性探测，尝试找出失效槽
            for (int i = prevIndex(staleSlot, len);// 入参槽位的前一个开始扫描
                 (e = tab[i]) != null;    // 扫描到空槽，循环终止；否则一直循环
                 i = prevIndex(i, len))   // 环形意义的前一个
                if (e.get() == null) // 遇到失效槽，标记它准备清理
                    slotToExpunge = i;


            // 若跳出循环，来到这里，表示存在空槽
            // 待清理槽位可能是入参槽位，可能是其他槽位

            
            // 向后线性探测：尝试找出其他槽有同样 key
            for (int i = nextIndex(staleSlot, len);// 入参槽位的后一个开始扫描
                 (e = tab[i]) != null;    // 扫描到空槽，循环终止；否则一直循环
                 i = nextIndex(i, len)) { // 环形意义的后一个
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value; //替换 value

                    tab[i] = tab[staleSlot];// 入参槽位 和 i槽位 交换
                    tab[staleSlot] = e;
                    
                    // 交换后，入参 key-value 终于放进了入参槽位

                    if (slotToExpunge == staleSlot) // 向前线性探测没找到失效槽位
                        slotToExpunge = i; // 交换后的 i槽位 是失效槽

                    // 对标记的失效槽进行清理
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }// end if (k == key)
                // 执行到此，则说明（k != key）or（k == null）
                // 而（k == null）又说明 i槽位 是失效槽

                // 当前槽位为失效槽 && 向前线性探测没找到失效槽位
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i; 
            }// end 向后线性探测

            // 来到这里，说明没找到 key 值，
            // 仍然把入参 key-value 放进入参槽位

            tab[staleSlot].value = null; //便于 GC 回收
            tab[staleSlot] = new Entry(key, value);

            // 若存在其他失效槽位，则清理它
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);

            // 整个 replaceStaleEntry 函数看下来，可以简单总结：
            // 向前探测，尝试找出失效槽位
            // 向后探测，尝试找出其他槽有同样 key
            // 若找到，替换其value为入参值，并做交换到入参槽位
            // 找不到，直接在入参槽位放一个新entry，带上入参 key-value
            // 从结果看，入参 key-value 是一定放到入参槽的，之后会尽可能的尝试清理其他失效槽位
        }
        
        ......
```

<br>

##### expungeStaleEntry(int i) 函数

进入 expungeStaleEntry(int i) 函数源码：<br>

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
    
        ......
        
        /**
         * 清理连续段
         * @param staleSlot 清理从 staleSlot 开始的一个连续段
         * @return 一个连续段后空槽的下标
         */
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) { // 失效槽则清理
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1); // 重新 hash计算下标
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e; // 向后挪到一个空槽
                    }
                }
            }
            return i;// 返回空槽的下标
        }
        
        ......
```

<br>

##### cleanSomecSlots() 函数

进入 cleanSomecSlots() 函数源码：<br>

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
    
        ......    

        /**
         * 启发式清理
         * @param i 从 i 槽开始向后探测，即扫描
         * @param n 控制扫描次数
         * @return 是否有清理
         */
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) { // 失效槽
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);// 清理一个连续段，返回空槽的下标
                }
            } while ( (n >>>= 1) != 0);// log n次扫描没有发现无效slot，函数就结束了
            return removed;
        }
        
        ......
```

<br>

##### rehash() 函数

接着再看 rehash() 函数的实现，源码：<br>

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
    
        ......  
        
        /**
         * 全量清理并扩容
         * 仅在 set 函数中调用
         */
        private void rehash() {
            expungeStaleEntries(); // 全量清理

            // threshold - threshold / 4 = threshold 3/4 = len2/3 * 3/4 = len 1/2
            if (size >= threshold - threshold / 4) // 等价于 size >= len 1/2
                resize(); // 扩容为原来的2倍
        }
```

<br>

##### expungeStaleEntries() 函数

接着再看 expungeStaleEntries() 函数的实现，源码：<br>

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
    
        ...... 
        
        /**
         * Expunge all stale entries in the table.
         */
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
```

<br>

##### resize() 函数

接着再看 resize() 函数的实现，源码：<br>

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
    
        ...... 
        
        /**
         * 扩容为原来的2倍
         * 扩容后，原 entry 放入新 table 时，
         */
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) { // 失效槽
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e; // 向后挪到一个空槽
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```

<br>
到此 set(...) 函数源码分析结束，引用参考文章的一段话来做总结：<br>

> 我们来回顾一下ThreadLocal的set方法可能会有的情况

> 探测过程中slot都不无效，并且顺利找到key所在的slot，直接替换即可
探测过程中发现有无效slot，调用replaceStaleEntry，效果是最终一定会把key和value放在这个slot，并且会尽可能清理无效slot
在replaceStaleEntry过程中，如果找到了key，则做一个swap把它放到那个无效slot中，value置为新值
在replaceStaleEntry过程中，没有找到key，直接在无效slot原地放entry
探测没有发现key，则在连续段末尾的后一个空位置放上entry，这也是线性探测法的一部分。放完后，做一次启发式清理，如果没清理出去key，并且当前table大小已经超过阈值了，则做一次rehash，rehash函数会调用一次全量清理slot方法也即expungeStaleEntries，如果完了之后table大小超过了threshold - threshold / 4，则进行扩容2倍

<br>

在次贴上参考文章链接：[大神级的细节，表述和逻辑：ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html) <br>
建议去该链接文章细读！

<br>

#### 取的实现

<br>

##### get() 函数

get() 函数用到的子函数，都在 set(...) 函数分析时遇到过。

先看 get() 函数源码：

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
    
        ...... 
        
        /**
         * key 经过hash计算出下标 i，
         * 若 i槽位没找到，
         * 调用 getEntryAfterMiss() 向后线性探测
         */ 
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
        
        ...... 
```

<br>

##### getEntryAfterMiss(...) 函数

再看 getEntryAfterMiss(...) 函数源码:

```
// .../Android/sdk/sources/android-28/java/lang/
// ThreadLocal.java

    static class ThreadLocalMap {
    
        ...... 
        
        /**
         * 向后线性探测，顺便清理失效槽；
         * 找到同 key 就返回节点，找不到就返回 null
         */ 
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
        
        ...... 
```

<br>

### 总结

ThreadLocal 的工作原理，简单的说就是做线程隔离的存储操作；<br>
首先通过 Thread.currentThread() 这个函数可以获取当前的线程，<br>
然后把要存储的数据对象，存到当前线程的 ThreadLocalMap 中；

关于 ThreadLocal.ThreadLocalMap 的基本工作原理，
重分析了三点：

- 存储结构
    - 弱引用派生类，将引用当作 key；内有一个 Object 类成员变量保存 value。

- 存储槽位的确定
    - 魔数累加，(2的幂)取模，产生均匀分布的槽位 

- 存取的实现
    - 数组 table 存储，单元存储结构如第一点所讲，存储位置如第二点所讲 
    
    - 这里存取的特别之处在于，作为弱引用的key，可能会被回收，产生失效槽
    
    - 无论 set 还是 get，都会尽量清理失效槽，采用线性探测，环形移动下标等方式
    
    - expungeStaleEntry(int i) 就是核心的清理函数，会清理 i 开始的一个连续段；其他清理函数的内部都会调用该函数
    
    - 触发清理的时机：
        - set 和 get 操作的扫描到失效槽
        - set 操作把数据填入新槽位后
        
    - 扩容时机：
        - set 操作把数据填入新槽位后，没有清理 && 存储个数 >= 阈值

<br>

**最后**

感谢该链接：[ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html) 的文章和作者，提供了非常细致的分析。<br>

写下本文章的目的在于自己走一遍 ThreadLocal.ThreadLocalMap 的工作原理，加深印象，<br>
更多细节可以去参阅读该链接：[ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html) 的文章。