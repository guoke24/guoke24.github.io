---
layout:     post
title:      "View 工作原理 layout 流程"
subtitle:   "read the fxxking source code"
date:       2020-02-01 12:00:00
author:     "GuoHao"
header-img: "img/waittouse.jpg"
catalog: true
tags:
    - Android
    - 源码
    - View
---

# 前言
参考文章： 
[凶残的程序员-View 的工作流程]( https://blog.csdn.net/qian520ao/article/details/78657084 )

[Android 源码分析 - View的measure、layout、draw三大流程]( https://www.jianshu.com/p/aa3b1f9717b7 )

[深入理解MeasureSpec]( https://www.jianshu.com/p/a790982fd20e )

本文章的源码，可以去该链接 [ Android9.0 - API28 ](https://www.androidos.net.cn/android/9.0.0_r8/xref)  直接搜索，浏览。

或者 AndroidStudio 查看 sdk 中 /sdk/sources/android-28/android 路径下的源码也是一样的。

# View的三大工作流程  

ActivityThread#handleResumeActivity 函数<br>
创建 ViewRootImpl 实例，DecorView 实例的引用传递给其内部的 mView 变量，然后执行 ViewRootImpl#requestLayout 函数，开始 View 的工作流程。<br>
requestLayout 函数，会触发 View 工作的三大流程：measure，layout，draw。<br>
调用链：<br>
--> ActivityThread#handleResumeActivity ...... <br>
--> ViewRootImpl#setView<br>
--> ViewRootImpl#requestLayout<br>
--> ViewRootImpl#scheduleTraversals<br>
--> mChoreographer.postCallback( ... , mTraversalRunnable , ...)<br>
--> ViewRootImpl#TraversalRunnable.run<br>
--> ViewRootImpl#doTraversal<br>
--> ViewRootImpl#performTraversals<br>
--> ViewRootImpl#performMeasure -> DecorView#measure<br>
--> ViewRootImpl#performLayout -> DecorView#layout<br>
--> ViewRootImpl#performDraw -> draw -> drawSoftware -> DecorView#draw<br>

---

# 第二大流程：布局
由 [[View的工作流程的源头]] 可知，一个 Activity 创建之后，通过该调用链：
- ActivityThread#handleResumeActivity
- ViewRootImpl#requestLayout
- ViewRootImpl#performTraversals
- ViewRootImpl#performMeasure -> DecorView#measure
- ViewRootImpl#performLayout -> DecorView#layout
先执行了测量流程，等测量流程执行完之后，每个 View 都完成了对自己的测量，测量结果保存在 mMeasuredWidth，mMeasuredHeight  变量。
测出每个 View 的大小之后，接着就是执行布局流程，确定每个 View 的位置了。

# 步骤概览：
- 第零步，ViewRootImpl#performTraversals
  - 若请求了绘制，并且 「窗口活动」 或 「报告了下一次绘制」 ，就会执行 performLayout 函数

- 第一步，ViewRootImpl#performLayout
  - host.layout

- 第二步，DecorView-ViewGroup#layou
  - mTransition.layoutChange
  - super.layout -- View#layou
    - setFrame 设置自身布局
    - onLayout
    - listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);

- 第三步，DecorView#onLayout
  - super.onLayout -- FrameLayout#onLayout
    - FrameLayout#layoutChildren ：child.layout
  - 处理 outsets ？

- 第四步，普通ViewGroup的子类-View#layout
  - setFrame 设置自身布局
  - onLayout
  - listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);

- 第五步，普通ViewGroup的子类#onLayout
  - 根据自身布局特点，计算 child 的位置，child.layout 触发 child 的布局流程
  - LinearLayout 为例
    - LinearLayout#onLayout（负责根据不同布局，计算出子 View 位置）
    - LinearLayout#layoutVertical
    - for：LinearLayout#setChildFrame -- child.layout

- 第六步，普通View的子类-View#layout
  - setFrame 设置自身布局
  - onLayout
  - listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);

- 第七步，普通View的子类#onLayout
  - 以 TextView 为例
  -  TextView#onLayout
    - super.onLayout
    - 增加了bringPointIntoView 和 autoSizeText 两个功能

# 布局步骤源码详解：
## 第零步，ViewRootImpl#performTraversals
源码段0:
```
// ViewRootImpl 类

    private void performTraversals() {
        // cache mView since it is used so much below...
        final View host = mView;

	// 。。。

        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);

        if (didLayout) {
            performLayout(lp, mWidth, mHeight);
```
由此段代码可见，performLayout 的执行，受到 layoutRequested，mStopped 和 mReportNextDraw 三个变量的影响。追踪一下：

```
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
```
- mLayoutRequested 变量
在执行 requestLayout 函数时被设置为 true，说明请求了布局。
```
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```
- mStopped 变量默认值为 false
在执行 setWindowStopped 函数时被设置，该变量应该跟 Activity 的生命周期有关联。mStopped 为 false，说明该窗口时活动的。
```
    // Set to true if the owner of this window is in the stopped state,
    // so the window should no longer be active.
    boolean mStopped = false;

    void setWindowStopped(boolean stopped) {
        if (mStopped != stopped) {
            mStopped = stopped;
```
- mReportNextDraw 变量
从字面上看，表示报告了下一次绘制。
mReportNextDraw 在执行 reportNextDraw 函数时被设置为 true。
```
    private void reportNextDraw() {
        if (mReportNextDraw == false) {
            drawPending();
        }
        mReportNextDraw = true;
    }
```
- 综合以上，我们可以得到一个简单且不严谨的结论：
只有当请求了绘制（mLayoutRequested == true），
而且 该窗口处于活动状态（mStopped == false）或者 
报告了下一次绘制（mReportNextDraw == true）的情况下，才会执行 performLayout 函数。<br>


接着追踪一下 performLayout 函数的这三入参 lp, mWidth, mHeight，接着源码段0：
```
// ViewRootImpl 类

    private void performTraversals() {

        WindowManager.LayoutParams lp = mWindowAttributes;
        Rect frame = mWinFrame;

        if (mFirst || windowShouldResize ||insetsChanged ||viewVisibilityChanged || params != null ||mForceNextWindowRelayout) 
       {

	     // 。。。

            if (mWidth != frame.width() || mHeight != frame.height()) {
                mWidth = frame.width();
                mHeight = frame.height();
            }
```
- mWindowAttributes 变量
来自于 Activity 持有的 PhoneWindow 实例的 getAttributes() 函数，表示窗口的属性。见如下函数：ActivityThread#handleResumeActivity
```
	View decor = r.window.getDecorView();
	// 略。。。
	ViewManager wm = a.getWindowManager();
	WindowManager.LayoutParams l = r.window.getAttributes();
	// 略。。。
	wm.addView(decor, l);
```
- mWinFrame 变量
保存 WindowManagerService 修改后的Activity窗口的宽度和高度。
引用参考该链接：https://blog.csdn.net/wangaiji/article/details/98210394
>注意！ViewRoot类的另外两个成员变量mWidth和mHeight也是用来描述Activity窗口当前的宽度和高度的，但是它们的值是由应用程序进程上一次主动请求WindowManagerService服务计算得到的，并且会一直保持不变到应用程序进程下一次再请求WindowManagerService服务来重新计算为止。Activity窗口的当前宽度和高度有时候是被WindowManagerService服务主动请求应用程序进程修改的，修改后的值就会保存在ViewRoot类的成员变量mWinFrame中，它们可能会与ViewRoot类的成员变量mWidth和mHeight的值不同。
<br>
- 第零步小节
ViewRootImpl 的 performTraversals 中，主要有三个参数控制着 performLayout 是否被执行，若请求了绘制，并且在 「窗口活动」 或 「报告了下一次绘制」 两个条件满足一个的条件下，就会执行 performLayout 函数，传入的参数跟窗口的属性有关。


## 第一步，ViewRootImpl#performLayout
源码段1:
```
// ViewRootImpl 类

    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;

        final View host = mView;

        try {
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
```
比较简单的调用了 DecorView 的 layou 函数，流程继续。

## 第二步，DecorView-ViewGroup#layou
DecorView 类没有实现 layout 函数，直接使用父类 ViewGroup 的 layou 函数。
源码段2:
```
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
通知了观察者，然后调用 super.layout(l, t, r, b)；即 View#layou 函数。

### View#layou 源码
源码段2.1:
```
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
在 View#layou 函数的开头就非常直接的调用 setFrame 函数，设置布局的四个关键数值：mLeft，mTop，mBottom，mRight。setOpticalFrame 函数本质也是调用 setFrame 函数。

### View#setFrame 源码
代码段2.2：
```
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
            invalidate(sizeChanged);

            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
```
这四个入参，表示子控件对于父控件四个边的距离，在 View 类中用四个变量保存：mLeft，mTop，mBottom，mRight。<br>
再回到 View#layou 函数，执行完 setFrame 函数后，表示布局已经确定，接着调用 onLayout 函数，流程继续。

## 第三步，DecorView#onLayout

代码段3：
```
// DecorView 类

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
	// 即调用 FrameLayout 的 onLayout 函数
        super.onLayout(changed, left, top, right, bottom); 
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

代码段3.1：
```
//  FrameLayout 类

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }
```

代码段3.2：
```
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
layoutChildren 函数中，逐个计算好 子View 的布局，在调用 child.layout 触发 子 View 的布局流程。<br>
- 第三步小节
到这一步结束后，后续布局流程的 普通 ViewGroup的子类 和 普通 View 的子类 的 layout 和 onLayout 流程的大致相当的，都是调用 View#layout -- setFrame 设定自己的布局，ViewGroup的子类在 onLayout 函数中计算好 子View 的布局，并触发子 View 的布局流程。在这过程，ViewGroup的子类可能会调用到父类的一些函数。

## 第四步，普通ViewGroup的子类#layout
普通ViewGroup的子类，调用 ViewGroup#layou -- View#layout ，同代码段2.1。
 
## 第五步，普通ViewGroup的子类#onLayout
以 LinearLayout 为例，代码仅参考，不分析。
源码段5:
```
// LinearLayout 类

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
```

```
// LinearLayout 类

    /**
     * Position the children during a layout pass if the orientation of this
     * LinearLayout is set to {@link #VERTICAL}.
     *
     * @see #getOrientation()
     * @see #setOrientation(int)
     * @see #onLayout(boolean, int, int, int, int)
     * @param left
     * @param top
     * @param right
     * @param bottom
     */
    void layoutVertical(int left, int top, int right, int bottom) {
        final int paddingLeft = mPaddingLeft;

        int childTop;
        int childLeft;

        // Where right end of child should go
        final int width = right - left;
        int childRight = width - mPaddingRight;

        // Space available for child
        int childSpace = width - paddingLeft - mPaddingRight;

        final int count = getVirtualChildCount();

        final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
        final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

        switch (majorGravity) {
           case Gravity.BOTTOM:
               // mTotalLength contains the padding already
               childTop = mPaddingTop + bottom - top - mTotalLength;
               break;

               // mTotalLength contains the padding already
           case Gravity.CENTER_VERTICAL:
               childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
               break;

           case Gravity.TOP:
           default:
               childTop = mPaddingTop;
               break;
        }

        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();

                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();

                int gravity = lp.gravity;
                if (gravity < 0) {
                    gravity = minorGravity;
                }
                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                                + lp.leftMargin - lp.rightMargin;
                        break;

                    case Gravity.RIGHT:
                        childLeft = childRight - childWidth - lp.rightMargin;
                        break;

                    case Gravity.LEFT:
                    default:
                        childLeft = paddingLeft + lp.leftMargin;
                        break;
                }

                if (hasDividerBeforeChildAt(i)) {
                    childTop += mDividerHeight;
                }

                childTop += lp.topMargin;
		 //  计算好 child 的布局后，设置给 child
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

                i += getChildrenSkipCount(child, i);
            }
        }
    }
```

```
// LinearLayout 类

    private void setChildFrame(View child, int left, int top, int width, int height) {
        child.layout(left, top, left + width, top + height);
    }
```

## 第六步，普通View的子类-View#layout
普通View的子类，直接使用父类 View 的 layou 函数，设置了自己的布局，逻辑同第四步，也同源码段2.1。

## 第七步，普通View的子类#onLayout (以 TextView 为例)
自己的布局，在 layout 函数中已经设定了，onLayout 函数主要是拓展一些自己的特色功能的。
源码段7:
```
// TextView 类

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        if (mDeferScroll >= 0) {
            int curs = mDeferScroll;
            mDeferScroll = -1;
            bringPointIntoView(Math.min(curs, mText.length()));
        }
        // Call auto-size after the width and height have been calculated.
        autoSizeText();
    }
```

##### 最后附表格一张

|              | layout    | onLayout   |           |  layout  |  onLayout  |
|:----------   |:----------|:---------- |:----------|:----------|:----------|
|ViewGroup |super.layout；通知观察者| null |   View    |记录位置，调用onLayout |null|
|ViewGroup子类|null|根据自身布局的特点来确定布局，for循环 ：调用 child.layout|View子类|null|super.onLayout；拓展自己的功能|
