---
layout:     post
title:      "Activity 的根View - DecorView"
subtitle:   "Activity 界面是一个 ViewTree"
date:       2020-02-01 12:00:00
author:     "GuoHao"
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - Android
    - 源码
    - View
---


# 从一个简单例子开始
假设有一个 MyActivity extends Activity，
设置了布局 setContentView( R.layout.activity_my ); 
activity_my.xml 的源码如下：    
<br>
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".viewtest.MyActivity">
    
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" 
        android:text="MyActivity"/>

</LinearLayout>
```

已知一个 Activity 的根视图是 DecorView，那么 DecorView 内的视图结构是怎样的呢？

先说结论，结构如下：

```
第0层：DecorView
-第1层，第1个View：LinearLayout （screen_simple.xml）
--第2层，第1个View：ViewStub
--第2层，第2个View：FrameLayout（装载 activity_my.xml 的父容器，id 为 content）
---第3层，第1个View：LinearLayout（activity_my.xml）
----第4层，第1个View：TextView
```
其中，第3层的 LinearLayout，就是 activity_my.xml 的根视图；
第4层的 TextView，就是 TextView("MyActivity")
<br>
而第1层和第2层的视图，
对应 R.layout.screen_simple 即 screen_simple.xml 布局文件，位于：源码工程/framework/base/core/res/layout/，<br>
一个以 LinearLayout 为根的视图，带有两个子视图：ViewStub，FrameLayout。<br>

可以在此处源码找到答案：

```
// PhoneWinow 类
 protected ViewGroup generateLayout(DecorView decor) {

	//  。。。 省略
       
        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();

	// 加载布局，根据 features 来判断
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
            setCloseOnSwipeEnabled(true);
        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
		 //  。。。 省略 
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
		 //  。。。 省略
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
		 //  。。。 省略
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
		 //  。。。 省略
            } else {
                layoutResource = R.layout.screen_title;
            }
            // System.out.println("Title!");
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            // Embedded, so no decoration is needed.
            layoutResource = R.layout.screen_simple;
            // System.out.println("Simple!");
        }

        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

	ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

	//  。。。 省略

	return contentParent;
    }
```
上述加载的布局文件都是系统内置的，常用的布局文件是这两个：
R.layout.screen_title 和 R.layout.screen_simple，都位于 framework/base/core/res/layout/
<br>
R.layout.screen_simple 的源码：
<br>
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```
一个 LinearLayout 的根视图，带有两个子视图：ViewStub，FrameLayout。
而 FrameLayout 就是 装载 activity_my.xml 的父容器，id 为 content，
其引用由 PhoneWinow#generateLayout 函数创建并返回给变量 mContentParent，
并通过 PhoneWinow#setContent 函数中的语句 ：
```
	mLayoutInflater.inflate(layoutResID, mContentParent); 
```
把上述的 activity_my.xml  设置给 mContentParent。

<br>
接下来理一下，从 Activity 的 setContentView 函数，是如何构建 DecorView 和其子视图的。
# 创建 DecorView 的调用链

- Activity#setContentView
```
	getWindow().setContentView(layoutResID);
```

- PhoneWinow#setContent
```
        if (mContentParent == null) {
            installDecor();

	if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
	    // 。。。
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
```
- PhoneWinow#installDecor
```
        if (mDecor == null) {
	    // 生成 DecorView
            mDecor = generateDecor(-1);

	// 。。。

        if (mContentParent == null) {
            // R.layout.screen_simple 设置为 mDecor
	    // id 为 content 的视图引用交给 mContentParent
            mContentParent = generateLayout(mDecor);
```
- PhoneWinow#generateDecor ( return new DecorView )
```
	return new DecorView(context, featureId, this, getAttributes());
```
- PhoneWinow#generateLayout
```
	layoutResource = R.layout.screen_simple;
	mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
	ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
	return contentParent;
```

详细的讲解，参考：
[该文章，对 DecorView 区域划分有讲解的](https://blog.csdn.net/nihaomabmt/article/details/85248700)

