---
layout:     post
title:      "View 工作原理 measure 流程"
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

# 背景知识

Android 是一个操作系统，也是一个平台，对于 Android APP 开发者来说，Android 是一个大框架。Android OS 是一个运行在移动终端的操作系统，其触摸操作是人机交互中的一大特点，因而对于 APP 开发者来说，如何构建出优雅美观且人性化的UI界面是非常重要的一项技能。<br>

想要学习如何做出一些好看的UI界面，对 View 的工作原理的了解是必不可少的。  
View 的工作原理，在网上看了很多讲解的文章，也看了很多源码，但有句老话：纸上得来终觉浅，绝知此事要躬行。故而行动起来，写下这篇文章，梳理了 View 的工作原理的相关知识点，留作以后参考。<br>

在写该文章之前，我一直在想，文章要以怎样的结构，才能比较流畅，清晰的阐述那些复杂又嵌套包含的知识点呢？有时候，越想写得太清楚，就越难动手。不如用最朴素的方式，对于复杂的知识点梳理，采取总分的结构，先总结再展开的方式来行文。于是，总分结构将是本文的框架。


# 总览

## View 和 ViewGroup 的概念

View 是一个视图，也是一个控件，它是展示某种类型内容的单体，是构成复杂界面的基本单位。View 可以是一个标签，一个输入框或一张图片等等。一个复杂界面，会有多个 View 构成，那么这个时候，就出现了分组归纳的需求，古人云，格物致知，那放到这里就是需要一个 View 的容器，以便将 View 分组，这个 View 的容器，就是 ViewGroup。ViewGroup 是 View 的子类，ViewGroup 可以容纳 View 和 ViewGroup。一个复杂界面，其中的多个 View 和 ViewGroup 会组成一个树形的结构体，根是一个 ViewGroup，最末端的枝叶，可以是 View，也可以是 ViewGroup。我们把这样的树形结构体，称为 ViewTree。 

## View 依赖的环境

就像 Android OS 需要依赖硬件终端才能运行，ViewTree 也需要要放入特定的环境，才能工作，那就是 Activity 中的 PhoneWindow 实例，其内部变量 mDecor 会引用 ViewTree 的根 View，即 DecorView 实例。参考文章 [[Activity 的根View - DecorView]] 和 [[View的工作流程的源头]]，可知详细内容。 

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

# 第一大流程：测量
由 [[View的工作流程的源头]] 可知，一个 Activity 创建之后，通过该调用链：
- ActivityThread#handleResumeActivity
- ViewRootImpl#requestLayout
- ViewRootImpl#performTraversals
- ViewRootImpl#performMeasure -> DecorView#measure
触发了 Activity 的根View -- DecorView 的测量流程，
同时也是触发了该 Activity 整个 ViewTree 的测量流程。

# 步骤概览：

- 第零步，ViewRootImpl#performTraversals
  - 调用 getRootMeasureSpec 测量 DecorView 的 MeasureSpace
  - 调用 performMeasure

- 第一步，ViewRootImpl#performMeasure
  - 调用 DecorView#measure

- 第二步，DecorView-View#measure
  - 判断是否需要测量，调用 onMeasure

- 第三步，DecorView#onMeasure
  - 强制设置 精确模式
  - 调用 FrameLayout#onMeasure
    - for 循环：调用 GroupView#measureChildWithMargins
```
	getChildMeasureSpec 测量 child 的 宽/高MeasureSpec
	child.measure( childWidthMeasureSpec, childHeightMeasureSpec )
```
    - 测完child，再测自己，View#setMeasuredDimension

- 第四步，普通ViewGroup-View#measure（LinearLayout 为例）
  - 判断是否需要测量，调用 onMeasure

- 第五步，普通ViewGroup的子类#onMeasure（LinearLayout 为例）
  - 根据布局特点，做特定的测量
    - 采用后根序的方式，先遍历测量 children，再测量自己
  - 例如：LinearLayout#onMeasure，
    - 调用 垂直 或 水平 布局的测量函数
```
	第一次遍历 children ，记录最大值
	第二次遍历 children ，调用 child.measure，再次记录最大值
	最后再测量自己，调用 setMeasuredDimension
```

- 第六步，普通View的子类-View#measure（TextView 为例）
  - 判断是否需要测量，调用 onMeasure

- 第七步，普通View的子类-View#onMeasure（TextView 为例）
  - View#onMeasure 的实现
  - TextView#onMeasure 的实现

## MeasureSpec 的简单理解
MeasureSpec 首先是 View 类的内部类，称为测量规格，简单的说是一个父类给子类的约束。<br>
MeasureSpec 用一个32位的 int 值表示，高2位表示 SpecMode，低30位表示 SpecSize。<br>
MeasureSpec类内提供的关键函数： <br>
- makeMeasureSpec (@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                          @MeasureSpecMode int mode)
```
该函数可以把接收独立的 SpecSize 和 SpecMode 作为参数， 合成一个 32 位的 int 类型的 MeasureSpec 值并返回。
```
<br>
- getMode (int measureSpec)
```
从一个 32 位的 int 类型的 MeasureSpec 值提取 SpecMode 值并返回。
```
<br>
- getSize (int measureSpec)
```
从一个 32 位的 int 类型的 MeasureSpec 值提取 SpecSize 值并返回。
```
有了这几个函数，MeasureSpec 的计算得到了规范统一。
后续源码分析中，设计测量的都用到这几个函数。<br>

# 测量步骤源码详解：

## 第零步，ViewRootImpl#performTraversals 源码
- 调用 getRootMeasureSpec 测量 DecorView 的 MeasureSpace
- 调用 performMeasure

代码段0：
源码来源于：[该链接](https://www.androidos.net.cn/android/9.0.0_r8/xref//frameworks/base/core/java/android/view/ViewRootImpl.java) <br>
```
// ViewRootImpl 类

    private void performTraversals() {
        // cache mView since it is used so much below...
        final View host = mView;

	// 2832 line
	int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width); // 0-1
	int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

	// 2844 line
	 performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); // 0-2
	
```
注释0-1处，调用 getRootMeasureSpec 函数，测量 DecorView 的宽高 MeasureSpec ;
注释0-2处，调用 performMeasure 函数 ，传入0-1处的测量结果，继续流程。<br>

- 继续 0-1 处，getRootMeasureSpec 函数
### getRootMeasureSpec 函数
代码段0.1：
```
// ViewRootImpl 类

    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```
此处可以插入一个表格说明，列出三种种情况。暂略。。。

执行该函数得到的宽高 MeasureSpec，传入 0-2 处的 performMeasure 函数，流程继续。<br>

## 第一步，ViewRootImpl#performMeasure 源码
- 调用 DecorView#measure

代码段1:
源码来源于：[该链接](https://www.androidos.net.cn/android/9.0.0_r8/xref//frameworks/base/core/java/android/view/ViewRootImpl.java) <br>
```
// ViewRootImpl 类

    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            // 1-1
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec); 
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

```
在注释 1-1 处，调用 DecorView 的 measure 函数，流程继续。

## 第二步，DecorView-View#measure 源码
- 判断是否需要测量，调用 onMeasure
DecorView 是 View 的子类，View 的 measure 函数是 final 修饰的，任何子类都必须直接使用 View 的 measure 函数，不能重写。<br>

代码段2:
代码来源于：/sdk/sources/android-28/android/view/View.java<br>
```
// View 类：

    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
                int oHeight = insets.top  + insets.bottom;
		// 2-1，可能需要调整一下，使其不为负数
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

	   // 。。。
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) { // 2-2 判断是否需要测量

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                     onMeasure(widthMeasureSpec, heightMeasureSpec);  // 2-3
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            }

		// 。。。
        }

         mOldWidthMeasureSpec = widthMeasureSpec; // 记录旧规格
        mOldHeightMeasureSpec = heightMeasureSpec;
    }
```
在 2-2 处，判断是否需要测量；
在 2-3 处，调用 onMeasure 函数，其参数 widthMeasureSpec 和  heightMeasureSpec，就是入参的两个函数 ，有可能调整过，有可能原封不动。
流程继续。

## 第三步，DecorView#onMeasure 源码
- 强制设置 精确模式
- 调用 FrameLayout#onMeasure
  - for 循环：调用 ViewGroup#measureChildWithMargins
    - getChildMeasureSpec 测量 child 的 宽/高MeasureSpec
    - child.measure( childWidthMeasureSpec, childHeightMeasureSpec )
  - 测完child，再测自己，View#setMeasuredDimension

代码段3：
源码来源于：[该链接](https://www.androidos.net.cn/android/9.0.0_r8/xref//frameworks/base/core/java/com/android/internal/policy/DecorView.java)<br>
```
// DecorView 类

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

        final int widthMode = getMode(widthMeasureSpec);
        final int heightMode = getMode(heightMeasureSpec);

        boolean fixedWidth = false;
        mApplyFloatingHorizontalInsets = false;
        if (widthMode == AT_MOST) {
            final TypedValue tvw = isPortrait ? mWindow.mFixedWidthMinor : mWindow.mFixedWidthMajor;
            if (tvw != null && tvw.type != TypedValue.TYPE_NULL) {
                final int w;
                if (tvw.type == TypedValue.TYPE_DIMENSION) {
                    w = (int) tvw.getDimension(metrics);
                } else if (tvw.type == TypedValue.TYPE_FRACTION) {
                    w = (int) tvw.getFraction(metrics.widthPixels, metrics.widthPixels);
                } else {
                    w = 0;
                }
                if (DEBUG_MEASURE) Log.d(mLogTag, "Fixed width: " + w);
                final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
                if (w > 0) {
                    widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            Math.min(w, widthSize), EXACTLY);
                    fixedWidth = true;
                } else {
                    widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            widthSize - mFloatingInsets.left - mFloatingInsets.right,
                            AT_MOST);
                    mApplyFloatingHorizontalInsets = true;
                }
            }
        }

	// heightMode == AT_MOST， 跟 widthMode 一样的处理逻辑
        // ...

        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        // ...
    }
```
在 DecorView 的 onMeasure 函数中，若 widthMode == AT_MOST，强制将其规格模式设置为 EXACTLY，然后调用父类 FrameLayout 的 onMeasure 函数。
### FrameLayout#onMeasure 源码
代码段3.1
源码来源于：[该链接]( https://www.androidos.net.cn/android/9.0.0_r8/xref//frameworks/base/core/java/android/widget/FrameLayout.java )<br>
```
// FrameLayout 类

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                // 3.1-1 测量 child 的 MeasureSpec，
                // 并调用 child.measure，开启 child 的测量流程
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
		// 。。。
            }
        }

	// 。。。

        // 3.1-2，调用基类 View 类的 setMeasuredDimension 函数，设置大小
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

    }
```

接着 3.1-1 处，measureChildWithMargins 函数是在 ViewGroup 中定义的，看源码：
### ViewGroup#measureChildWithMargins 源码
代码段3.1.1：
```
// ViewGroup 类

    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
其内部又调用了 getChildMeasureSpec 函数。

### ViewGroup#getChildMeasureSpec 源码
代码段3.1.1.1：
代码源于：/sdk/sources/android-28/android/view/ViewGroup.java
```
// ViewGroup 类

    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
此处应放一个表格，列出九种情况。暂略。。。


回到代码段3.1，看注释 3.1-2 处，调用 View 类的 setMeasuredDimension 函数。
### View#setMeasuredDimension 源码
代码段3.1.2：
代码来源于：/sdk/sources/android-28/android/view/View.java<br>
```
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
```

```
    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```


## 第四步，普通ViewGroup-View#measure
- 判断是否需要测量，调用 onMeasure
View 的 measure 函数是 final 修饰的，所以 VIew 子类调用 measure 函数时，只能调用 View#measure
跟代码段2的一致，此处省略。

## 第五步，普通ViewGroup的子类#onMeasure（LinearLayout 为例）
ViewGroup 类本身的 onMeasure 是空的，留给子类去实现，以满足不同布局的测量需求。
ViewGroup 的子类一般做的事：<br>
- 根据布局特点，做特定的测量
  - 采用后根序的方式，先遍历测量 children，再测量自己
- 例如：LinearLayout#onMeasure，
  - 调用 垂直 或 水平 布局的测量函数
    - 第一次遍历 children ，记录最大值
    - 第二次遍历 children ，调用 child.measure，再次记录最大值
    - 最后再测量自己，调用 setMeasuredDimension

### LinearLayout#onMeasure 源码
代码段5：
代码源于：/sdk/sources/android-28/android/widget/LinearLayout.java
```
// LinearLayout 类

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if (mOrientation == VERTICAL) {
            measureVertical(widthMeasureSpec, heightMeasureSpec);
        } else {
            measureHorizontal(widthMeasureSpec, heightMeasureSpec);
        }
    }
```
### LinearLayout#measureVertical 源码
```
// LinearLayout 类

	// 第一遍遍历 children，记录宽度的最大值 maxWidth
        for (int i = 0; i < count; ++i) {
		//。。。
		final int margin = lp.leftMargin + lp.rightMargin;
		final int measuredWidth = child.getMeasuredWidth() + margin;
		maxWidth = Math.max(maxWidth, measuredWidth);
		//。。。
	}

	// 第二遍遍历 children，调用 child.measure，并更新宽度的最大值 maxWidth
	 for (int i = 0; i < count; ++i) {
                final View child = getVirtualChildAt(i);
                if (child == null || child.getVisibility() == View.GONE) {
                    continue;
                }

                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                final float childWeight = lp.weight;
                if (childWeight > 0) {

                    final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            Math.max(0, childHeight), MeasureSpec.EXACTLY);
                    // getChildMeasureSpec 函数源码，同代码段3.1.1.1
                    final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,
                            lp.width);
                    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

                }

                final int margin =  lp.leftMargin + lp.rightMargin;
                final int measuredWidth = child.getMeasuredWidth() + margin;
                maxWidth = Math.max(maxWidth, measuredWidth);
	}

	
	maxWidth += mPaddingLeft + mPaddingRight;
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
	// 测完child，再测自己，setMeasuredDimension 源码，同代码段3.1.2
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                heightSizeAndState);
```

## 第六步，普通View的子类-View#measure 源码
- 判断是否需要测量，调用 onMeasure
View 的 measure 函数是 final 修饰的，所以 VIew 子类调用 measure 函数时，只能调用 View#measure
跟代码段2的一致，此处省略。

## 第七步，普通View的子类-View#onMeasure
### View#onMeasure 源码
代码段7:
```
// View 类：

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }

    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```

#### View#getSuggestedMinimumWidth 源码
```
// View 类：

    // 获得 mMinWidth 和 背景最小宽度之间的最大值
    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

#### View#getDefaultSize 源码
```
// View 类：

    // 获得纯尺寸，而不是测量规格
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

- 普通的 View 子类，几乎都会重写 View#onMeasure 函数，自己进行一番处理后，再调用 setMeasuredDimension 设置结果。

- 例子：TextView#onMeasure 的实现
### TextView#onMeasure 源码
```
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int width;
        int height;

	// 。。。

        if (widthMode == MeasureSpec.EXACTLY) {
            // Parent has told us how big to be. So be it.
            width = widthSize;
        } else {

	    //  进行一番判断比较

            // Check against our minimum width
            width = Math.max(width, getSuggestedMinimumWidth());

            if (widthMode == MeasureSpec.AT_MOST) {
                width = Math.min(widthSize, width);
            }
        }

	// 。。。

        if (heightMode == MeasureSpec.EXACTLY) {
            // Parent has told us how big to be. So be it.
            height = heightSize;
            mDesiredHeightAtMeasure = -1;
        } else {
            int desired = getDesiredHeight();

            height = desired;
            mDesiredHeightAtMeasure = desired;

            if (heightMode == MeasureSpec.AT_MOST) {
                height = Math.min(desired, heightSize);
            }
        }

	// 。。。

	// 最终结果保存在变量 width, height，并设置 
        setMeasuredDimension(width, height);
    }
```
### 最后总结

借用该链接：[凶残的程序员-View 的工作流程] ( https://blog.csdn.net/qian520ao/article/details/78657084 ) 的一张图，来表示大致的工作流程。
![](https://upload-images.jianshu.io/upload_images/9124992-af00916cc0d4414d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

再来一张表格，展示 measure 函数 和 onMeasure 函数在不同类实现上的一些区别：

|                 | measure    | onMeasure   |           |  measure  |  onMeasure  |
|:----------   |:----------       |:----------          |:--------|:----------      |:----------          |
|ViewGroup |null，调用View#measure| null |   View    |判断是否要测量，调用 onMeasure |默认的设置尺寸逻辑|
|ViewGroup子类|null|根据自身布局的特点来测量子 View，其中会调用到 ViewGroup 的 measureChildWithMargins 函数 和 getChildMeasureSpec 函数；测出子 View 的规格后。for循环：调用 child.measure 触发子 View 的测量流程，并传递规格给子 View|View子类|null|完全重写，根据自身特点来测量，调用View#setMeasuredDimension来保存测量尺寸|



