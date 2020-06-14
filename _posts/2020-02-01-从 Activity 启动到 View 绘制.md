---
layout:     post
title:      "从 Activtity 启动到 View 绘制"
subtitle:   "onResume 后绘制 ViewTree"
date:       2020-02-01 12:00:00
author:     "GuoHao"
header-img: "img/guitar.jpg"
catalog: true
tags:
    - Android
    - 源码
    - View

---

## 时序图

### Activtity 的第一次绘制

![](/img/Activtity 的第一次绘制.png)

Activity 的 onCreate 中会通过 setContentView 函数添加自定义的布局文件到 Activity 所关联的 ViewTree，完成构造 ViewTree ；

然后，在 Activity 的 onResume 函数以后，开始绘制 Activity 所关联的 ViewTree；

<br>

### 绘制工作的三大流程

每一次绘制都是 post 一个任务让 handler 处理：

![](/img/绘制工作的三大流程.png)

[ViewRootImpl.java 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/view/ViewRootImpl.java)

[查源码的链接](https://www.androidos.net.cn/androidossearch?query=&sid=&from=code)

## 简单源码

### Activtity 的第一次绘制

- ActivityThread#handleResumeActivity

```
	View decor = r.window.getDecorView();
	// 略。。。
	ViewManager wm = a.getWindowManager();
	WindowManager.LayoutParams l = r.window.getAttributes();
	// 略。。。
	wm.addView(decor, l);
```

- WindowManagerImpl#addView
- WindowManagerGlobal#addView

```
	root = new ViewRootImpl(view.getContext(), display);
	// 略。。。
	try {
	       root.setView(view, wparams, panelParentView);
    }
```

一个 Activity 对应 一个 window，一个 DecorView 和 一个 ViewRootImpl；

- ViewRootImpl#setView

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
      synchronized (this) {
	if (mView == null) {
	    mView = view;

	    mWindowAttributes.copyFrom(attrs);
	    // 略。。。
	    // Schedule the first layout -before- adding to the window
	    // manager, to make sure we do the relayout before receiving
	    // any other events from the system.
	    requestLayout();
```

- ViewRootImpl#requestLayout

```
	// 开始 View 的绘制工作流程
```

<br>
### 绘制工作的三大流程

- ViewRootImpl#requestLayout

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

- ViewRootImpl#scheduleTraversals

```
    mChoreographer = Choreographer.getInstance();

    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable/*接着这看*/, null);

            ......
        }
    }
```

- ViewRootImpl#mTraversalRunnable

```
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
```

- ViewRootImpl#doTraversal

```
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            ...

            performTraversals();// 此处触发三大流程 
	  
            ...
        }
    }
```

- ViewRootImpl#performTraversals

```
	boolean newSurface = false;
	WindowManager.LayoutParams lp = mWindowAttributes;
	Rect frame = mWinFrame;

	if (mWidth != frame.width() || mHeight != frame.height()) {
		mWidth = frame.width();
		mHeight = frame.height();
	}
	int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
	int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

	// ---------------------- 分割线 ----------------------

        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
        if (didLayout) {
            performLayout(lp, mWidth, mHeight);

	// ---------------------- 分割线 ----------------------

                if (!hadSurface) {
                    if (mSurface.isValid()) {
                        newSurface = true;

	// 略。。。
        if (!cancelDraw && !newSurface) {
            performDraw();

```

- ViewRootImpl#performMeasure

```
	mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
```

- ViewRootImpl#performLayout

```
	final View host = mView;

	host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

	mInLayout = false;
	int numViewsRequestingLayout = mLayoutRequesters.size();
	if (numViewsRequestingLayout > 0) {
		 // 。。。
	    ArrayList<View> validLayoutRequesters = getValidLayoutRequesters (mLayoutRequesters,false);

	     if (validLayoutRequesters != null) {
	       // 。。。
	       mInLayout = true;
	       host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
	       // 。。。
```

- ViewRootImpl#performDraw
- ViewRootImpl#draw
- ViewRootImpl#drawSoftware

```
	mView.draw(canvas);
```

<br>
## 拓展

### DecorView 的测量规格

在 ViewRootImpl#performTraversals 函数中的 <br>
`WindowManager.LayoutParams lp = mWindowAttributes;` <br>
决定了 DecorView 的布局规格，那么这个 mWindowAttributes 是在哪里决定了呢？

经过时序图，可以追踪到 ActiviyThread 的 handleResumeActivity 函数中：

```
    WindowManager.LayoutParams l = r.window.getAttributes();
```

r 就是 ActivityClientRecord（ ActivityThread 的内部类 ），其 window 字段的赋值：

```
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
            
        r.window = r.activity.getWindow();
```

```
// Activity.java
    private Window mWindow;
    
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
            
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        
        ...
        
    }
    
    public Window getWindow() {
        return mWindow;
    }
```

其实，`r.window.getAttributes()` 最终调用到的是 PhoneWindow 的 父类 Window 的 getAttributes() 函数；

```
// Window.java

    // The current window attributes.
    private final WindowManager.LayoutParams mWindowAttributes =
        new WindowManager.LayoutParams();
        
    public void setAttributes(WindowManager.LayoutParams a) {
        mWindowAttributes.copyFrom(a);
        dispatchWindowAttributesChanged(mWindowAttributes);
    }
    
    public final WindowManager.LayoutParams getAttributes() {
        return mWindowAttributes;
    }
```

```
// .../Android/sdk/sources/android-28/android/view/
// WindowManager.java

public interface WindowManager extends ViewManager {

    ...
    
    public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {

        ...
        
        public LayoutParams() {
            super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
            type = TYPE_APPLICATION;
            format = PixelFormat.OPAQUE;
        }
        ...
    }
    ...
}
```

```
// .../Android/sdk/sources/android-28/android/view/
// ViewGroup.java

public static class LayoutParams {
        ...
        public LayoutParams(int width, int height) {
            this.width = width;
            this.height = height;
        }
        ...
}
```

所以，DecorView 所在的 window 的宽高规格就是： <br>
LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT；

根据 getRootMeasureSpec 函数的逻辑：

|      | MATCH_PARENT | WRAP_CONTENT | default       |
|------|--------------|--------------|---------------|
| mode | EXACTLY      | AT_MOST      | EXACTLY       |
| size | windowSize   | windowSize   | rootDimension |

可知，**DecorView 的宽高的初始规格就是 EXACTLY & windowSize** ...

<br>
### Activity 与 Window 与 View 之间的关系

参考：[一篇文章看明白 Activity 与 Window 与 View 之间的关系](https://blog.csdn.net/freekiteyu/article/details/79408969)

引用图片：

<br>

#### onCreate() - Window 创建过程：

![onCreate() - Window 创建过程](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMzAxMTAyMjExNDkz)

<br>

#### onResume() - Window 显示过程：

![onResume() - Window 显示过程](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMzAxMTAyMjUyOTk1)

<br>

#### Activity 中 Window 创建过程：

![Activity 中 Window 创建过程](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMzAxMTAyMzE3ODMx)