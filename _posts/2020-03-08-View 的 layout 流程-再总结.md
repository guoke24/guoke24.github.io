---
layout:     post  
title:      "View 的 layout 流程-再总结"  
subtitle:   "计算 View 的位置"  
date:       2020-03-08 12:00:00  
author:     "GuoHao"  
header-img: "img/waittouse.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - View 

---

## 看图说话

### 流程图

借用该链接：[凶残的程序员-View 的工作流程]( https://blog.csdn.net/qian520ao/article/details/78657084 ) 的一张图，来表示大致的工作流程。

![](https://img-blog.csdn.net/20171130105250728?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcWlhbjUyMGFv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


<br>

### 时序图
来一张时序图：

![](/img/layout流程.png)

<br>

### 表格

表格对比

|              | layout    | onLayout   |           |  layout  |  onLayout  |
|:----------   |:----------|:---------- |:----------|:----------|:----------|
|ViewGroup |super.layout； 通知观察者| null |   View    |记录位置， 调onLayout |null|
|ViewGroup子类|null|根据自身布局的 特点来确定布局， for循环 ：调用 child.layout|View子类|null|super.onLayout； 拓展自己的功能|

<br>

## 简单源码梳理

### 第0步 performTraversals

- 0，ViewRootImpl#performTraversals [源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/view/ViewRootImpl.java)


```
// 代码段0：
// ViewRootImpl 类

private void performTraversals() {
    // cache mView since it is used so much below...
    final View host = mView;

    WindowManager.LayoutParams lp = mWindowAttributes;
    Rect frame = mWinFrame;

    if (mWidth != frame.width() || mHeight != frame.height()) {
        mWidth = frame.width();
        mHeight = frame.height();
    }

	final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
	if (didLayout) {
        performLayout(lp, mWidth, mHeight);
        ...
```

### 第1步 performLayout

- 1，ViewRootImpl#performLayout

```
// 源码段1:
// ViewRootImpl 类

    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;

        final View host = mView;

        try {
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
            ...
```

<br>
比较简单的调用了 DecorView 的 layou 函数，流程继续。

<br>

### 第2步 DecorView#layou

- 2，DecorView#layou

没有重写 layou 函数，直接使用父类 ViewGroup 的 layou 函数...

<br>

### 第3步 ViewGroup#layou

- 3，ViewGroup#layou

```
// 源码段2:
// ViewGroup 类

    @Override
    public final void layout(int l, int t, int r, int b) {
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);
            }
            super.layout(l, t, r, b);
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }
```

<br>
通知了观察者，然后调用 super.layout(l, t, r, b)；即 View#layou 函数。

<br>

### 第4步 View#layou

- 4，View#layou

```
// 源码段3:
// View类

    /**
     * Assign a size and position to a view and all of its
     * descendants
     *
     *
     * @param l Left position, relative to parent
     * @param t Top position, relative to parent
     * @param r Right position, relative to parent
     * @param b Bottom position, relative to parent
     */
    @SuppressWarnings({"unchecked"})
    public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);

	    // 。。。
	    
            // 通知观察者
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

    }
```

<br>
在 View#layou 函数的开头就非常直接的调用 setFrame 函数，设置布局的四个关键数值：mLeft，mTop，mBottom，mRight。setOpticalFrame 函数本质也是调用 setFrame 函数。

View#setFrame 源码<br>

```
// 代码段3.1：
// View类

    /**
     * Assign a size and position to this view.
     *
     * This is called from layout.
     *
     * @param left Left position, relative to parent
     * @param top Top position, relative to parent
     * @param right Right position, relative to parent
     * @param bottom Bottom position, relative to parent
     * @return true if the new size and position are different than the
     *         previous ones
     * {@hide}
     */
    protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;

        if (DBG) {
            Log.d(VIEW_LOG_TAG, this + " View.setFrame(" + left + "," + top + ","
                    + right + "," + bottom + ")");
        }

        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;

            // Remember our drawn bit
            int drawn = mPrivateFlags & PFLAG_DRAWN;

            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged); // 触发重绘

            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
```

<br>
这四个入参，表示子控件对于父控件四个边的距离，在 View 类中用四个变量保存：mLeft，mTop，mBottom，mRight。<br>
后续的流程，比如 onDraw 函数中，可以通过 getLeft()、getTop()、getBottom()、getRight() 这四个函数获取；

```
    public final int getLeft() {
        return mLeft;
    }
    
    public final int getRight() {
        return mRight;
    }
    
    ...
    
    public final int getWidth() {
        return mRight - mLeft;
    }
    
    public final int getHeight() {
        return mBottom - mTop;
    }
```

再回到 View#layou 函数，执行完 setFrame 函数后，表示布局已经确定，接着调用 onLayout 函数，流程继续。

<br>

### 第5步 DecorView#onLayout

- 5，DecorView#onLayout

```
// 代码段4
// DecorView 类

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        // 即调用 FrameLayout 的 onLayout 函数
        super.onLayout(changed, left, top, right, bottom); 
        
        // 其他一些处理逻辑
        getOutsets(mOutsets);
        if (mOutsets.left > 0) {
            offsetLeftAndRight(-mOutsets.left);
        }
        if (mOutsets.top > 0) {
            offsetTopAndBottom(-mOutsets.top);
        }
        if (mApplyFloatingVerticalInsets) {
            offsetTopAndBottom(mFloatingInsets.top);
        }
        if (mApplyFloatingHorizontalInsets) {
            offsetLeftAndRight(mFloatingInsets.left);
        }

        // If the application changed its SystemUI metrics, we might also have to adapt
        // our shadow elevation.
        updateElevation();
        mAllowUpdateElevation = true;

        if (changed && mResizeMode == RESIZE_MODE_DOCKED_DIVIDER) {
            getViewRootImpl().requestInvalidateRootRenderNode();
        }
    }
```

<br>
开头处即调用求父类 FrameLayout 的 onLayout 函数...

<br>

### 第6步 FrameLayout#onLayout

- 6，FrameLayout#onLayout

```
代码段5：
//  FrameLayout 类

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }
```

<br>

### 第7步 FrameLayout#layoutChildren

- 7，FrameLayout#layoutChildren

```
// 代码段6：
//  FrameLayout 类

    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        final int count = getChildCount();

        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();

                int childLeft;
                int childTop;

                int gravity = lp.gravity;
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }

                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT:
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }

                switch (verticalGravity) {
                    case Gravity.TOP:
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }

                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```

<br>
layoutChildren 函数中，逐个计算好 子View 的布局，在调用 child.layout 触发 子 View 的布局流程。<br>

<br>

### 第8步，child.layout

- 8，child.layout

从这一步开始，后续布局流程的 普通 ViewGroup的子类 和 普通 View 的子类 的 layout 和 onLayout 流程的大致相当的，都是调用 View#layout -- setFrame 设定自己的布局，ViewGroup的子类在 onLayout 函数中计算好 子View 的布局，并触发子 View 的布局流程。在这过程，ViewGroup的子类可能会调用到父类的一些函数。

<br>

## 总结

从 layout 这个流程看下来，发现它是一个先测自己，再测 子view 的流程；<br>

从第8步开始的 child.layout，child 也就是 DecorView 的第一个 子View，即系统布局 R.layout.screen_title 或 R.layout.screen_simple 或 其他类似的，统一放在源码工程的 frameworks/base/core/res/res/layout/ 路径下；

[R.layout.screen_title 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/res/res/layout/screen_title.xml) 和 [R.layout.screen_simple 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/res/res/layout/screen_simple.xml)

因为 R.layout.screen_title 这类的布局文件是 ViewGroup 类型的，所以第8步 child 就是它们的 **根view：LinearLayout**；

其实，这整个流程看下来，只是对 layout 流程有一个大概的了解，具体的View 子类和 ViewGroup 子类的 onLayout 函数都有自己的实现，需要去具体分析...

<br>

## 链接汇总

[R.layout.screen_title 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/res/res/layout/screen_title.xml) 

 [R.layout.screen_simple 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/res/res/layout/screen_simple.xml)

[ViewRootImpl 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/view/ViewRootImpl.java)

[DecorView 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref//frameworks/base/core/java/com/android/internal/policy/DecorView.java)

[FrameLayout 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref//frameworks/base/core/java/android/widget/FrameLayout.java)

[View 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/view/View.java)

[ViewGroup 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/view/ViewGroup.java)

[ViewStub extends View ](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/view/ViewStub.java)

[LinearLayout 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/widget/LinearLayout.java)

[TextView 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/widget/TextView.java)

[查源码的链接](https://www.androidos.net.cn/androidossearch?query=&sid=&from=code)
