---
layout:     post  
title:      "setContentView 的源码追踪"  
subtitle:   "onCreate 中构建 ViewTree"  
date:       2020-03-01 12:00:00  
author:     "GuoHao"  
header-img: "img/about-bg.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - View

---

## 参考

[View绘制体系（一）——从setContentView聊起](https://blog.csdn.net/boyeleven/article/details/82759753)

[View绘制体系（二）——View的inflate详解](https://blog.csdn.net/boyeleven/article/details/82776485)

## 前言

先从这张图说起：

引用自：[Android深入四大组件（七）Android8.0 根Activity启动过程（后篇）](http://liuwangshu.cn/framework/component/7-activity-start-2.html)

ActivityThread 启动 Activity 的过程：

![](https://s2.ax1x.com/2019/05/28/VeqcKU.png)

<br>
在 Activity 的 onCtreate 函数中，我们会调用：

```
    setContentView(R.layout.activity_xxx);
```

<br>
那么 setContentView(..) 的后续流程事如何的呢？请看接下来的分析...

<br>
## 正文

### setContentView 的时序图

![](/img/setContentView 的时序图.png)

<br>
setContentView 的关键步骤：

* 第4步 **installDecor()** 
    * 初始化 mDecor 和 mContentParent
* 第5步 **mLayoutInflater.inflate(...)** 
    * 解析 R.layout.xxx 布局文件，并添加到 mContentParent


问题：<br>
mContentParent 和 布局文件 R.layout.screensimple（ 即 screensimple.xml ），他们是什么关系？<br>
mContentParent 是 R.layout.screensimple 的根视图 ？<br>
mContentParent 是 R.layout.screensimple 的子 View ？<br>
mContentParent 是 R.layout.screensimple 的父容器 ？<br>

带着问题往下看...

### 关键步骤 installDecor

installDecor 是 PhoneWindow 类的函数，在 PhoneWindow 类的 setContentView 函数内被调用。

关于 Activity 的 setContentView 函数怎么调用到 PhoneWindow 类的 setContentView 函数，请参考该文章：[View绘制体系（一）——从setContentView聊起](https://blog.csdn.net/boyeleven/article/details/82759753)

#### PhoneWindow#setContentView 函数

PhoneWindow 类的 setContentView 函数[源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java)：

```
// PhoneWindow.java

    public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }
    
    ...
    
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            // 1，对应图中第4步
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            // 2，对应图中第5步
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

<br>
goin 第4步 `installDecor()`

#### installDecor 函数源码

```
// PhoneWindow.java

    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1); // 1，初始化 mDecor
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor); // 2，初始化 mContentParent
            
            // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
            mDecor.makeOptionalFitsSystemWindows();

            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);

            if (decorContentParent != null) {
                ...
            } else {
                mTitleView = findViewById(R.id.title);
                if (mTitleView != null) {
                    // 去掉标题栏 requestWindowFeature(Window.FEATURE_NO_TITLE);
                    // 在此处生效
                    if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                        final View titleContainer = findViewById(R.id.title_container);
                        if (titleContainer != null) {
                            titleContainer.setVisibility(View.GONE);
                        } else {
                            mTitleView.setVisibility(View.GONE);
                        }
                        mContentParent.setForeground(null);
                    } else {
                        mTitleView.setText(mTitle);
                    }
                }
            }
    }
```

<br>
插曲：[去掉标题栏参考该链接](https://blog.csdn.net/qq_42433597/article/details/93142531)

goin 注释1 `mDecor = generateDecor(-1);`

##### generateDecor 函数源码

```
// PhoneWindow.java

    protected DecorView generateDecor(int featureId) {
        // System process doesn't have application context and in that case we need to directly use
        // the context we have. Otherwise we want the application context, so we don't cling to the
        // activity.
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, getContext());
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        return new DecorView(context, featureId, this, getAttributes());
    }
```

<br>
末尾处，new DecorView 实例并返回。

接着看看 DecorView 的[源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/com/android/internal/policy/DecorView.java)：

##### DecorView 类的源码

```
// DecorView.java

public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
    private static final String TAG = "DecorView";
    ...
    private PhoneWindow mWindow;
    ViewGroup mContentRoot;
    ...
    void setWindow(PhoneWindow phoneWindow) {
        mWindow = phoneWindow;
        Context context = getContext();
        if (context instanceof DecorContext) {
            DecorContext decorContext = (DecorContext) context;
            decorContext.setPhoneWindow(mWindow);
        }
    }
    
    ...
    
    void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
    
        ...

        // 返回 layoutResource 所描述的 ViewTree 的根视图
        final View root = inflater.inflate(layoutResource, null);
        
        ...
        
        mContentRoot = (ViewGroup) root;
        initializeElevation();
    }
    
    ...
    
```

<br>
可知，DecorView 继承自 FrameLayout，内部持有 mWindow 和 mContentRoot 字段；

接着回到 installDecor 函数，

goin 注释2 `mContentParent = generateLayout(mDecor);`

##### generateLayout 的函数源码

```
// PhoneWinow 类

 protected ViewGroup generateLayout(DecorView decor) {

        ...
       
        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();

        // 加载布局，根据 features 来判断
        // 给 layoutResource 赋值
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            ...
        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            ...
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            ...
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
            ...
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                ...
            } else {
                // 1，默认情况
                layoutResource = R.layout.screen_title;
            }
            // System.out.println("Title!");
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            // 2，不带标题的情况
            // Embedded, so no decoration is needed.
            layoutResource = R.layout.screen_simple;
            // System.out.println("Simple!");
        }

        // 3，加载布局文件到 mDecor
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

        // 4，取出布局文件中的 id 为 content 的视图，并 return 给 mContentParent
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        
        // ID_ANDROID_CONTENT = com.android.internal.R.id.content;
        // 普通应用可以通过 Window.ID_ANDROID_CONTENT 来访问
        
        // findViewById 函数定义在 Window.java 
        // 其内部由调用 getDecorView().findViewById(id)
        // 接着会调用 ViewGroup 类的 findViewTraversal
        // 从顶层视图 mDecor 遍历所有子视图，找到就返回
        
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        //  。。。 省略

        return contentParent;
    }
```

<br>
插曲2：[Android基于Window.ID_ANDROID_CONTENT给定id添加子View](https://blog.csdn.net/zhangphil/article/details/76819848) <br>
插曲3：[findViewById的源码解析](https://www.jianshu.com/p/9ac12ed16b54)<br>
插曲4：[R.layout.screen_title 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/res/res/layout/screen_title.xml) 和 [R.layout.screen_simple 源码](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/res/res/layout/screen_simple.xml)
<br>

注释1，常规情况，会加载 R.layout.screen_title 这个布局文件;<br>

注释2，如果不带标题，则加载 R.layout.screen_simple 这个布局文件；<br>

screen_title 比 screen_simple 多了一个展示标题的 FrameLayout，该 FrameLayout 内含一个 id 为 “title” 的 TextView；

注释3，将 layoutResource 指向的布局文件加载，并将该布局文件的根视图赋值给 
mDecor 的 mContentRoot 字段， mDecor.onResourcesLoaded 具体函数可见上述的 DecorView 类的源码的 onResourcesLoaded 函数源码；<br>

注释4，取出布局文件中的 id 为 content 的视图，并 return 赋值给 mContentParent，见 installDecor 函数源码的注释2；

到此，走完了 installDecor 函数的关键流程，做一个小结...

<br>
##### installDecor 函数小结

installDecor 函数中，完成了顶层视图 mDecor 的创建，为其选择了一个布局文件并加载，大多是情况是 R.layout.screen_title 或 R.layout.screen_simple，
而 mContentParent 则为 mDecor 为根的视图树 ViewTree 下的一个 id 为 “content” 的 FrameLayout 类型的容器；

此时 Activity 的视图树结构如下：

```
第0层：DecorView
-第1层，第1个View：LinearLayout（ screen_title .xml 或 screen_simple.xml）
--第2层，第1个View：ViewStub（id 为 action_mode_bar_stub ）
--第2层，第1.5个View：FrameLayout（ screen_title .xml 才有）
---第三层，TextView：展示标题（ screen_title .xml 才有）
--第2层，第2个View：FrameLayout（id 为 content，即 mContentParent，装载 layout.xxx.xml 的父容器）
```

- （1层）DecorView
    - （2层）LinearLayout
        - （3层）ViewStub（id = action_mode_bar_stub ）
        - （3层）FrameLayout（ screen_title .xml 才有）
            - （4层）TextView（ screen_title .xml 才有，展示标题）
        - （3层）FrameLayout（id 为 content，即 mContentParent，装载 layout.xxx.xml 的父容器）

<br>
此处解决了开头的问题：<br>
mContentParent 和 布局文件 R.layout.screensimple（ 即 screensimple.xml ），他们是什么关系？<br>
答：mContentParent 是 R.layout.screensimple 的子 View <br>

**插一个问题**：mDecor 是 FrameLayout，也是 ViewGroup，ViewGroup 的子 View 都会 存放到 mChildren 这个 View[] 类型的字段中，但是在 DecorView 的源码没发现相关的操作，这是为何？

<br>
### 关键步骤 mLayoutInflater.inflate

接着 PhoneWindow 类的 setContentView 函数中的注释2，

```
    mLayoutInflater.inflate(layoutResID, mContentParent);
```


```
    private LayoutInflater mLayoutInflater;

    public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }
```

<br>
#### inflate 函数源码

[其源码链接](https://www.androidos.net.cn/androidossearch?query=LayoutInflater&sid=&from=code):

inflate 函数源码：

```
// LayoutInflater.java

    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        // 调用同名函数
        return inflate(resource, root, root != null);
    }
    
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            // 调用同名函数
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
    
    // 真正做事的函数
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                
                // 循环找到文档开头的标签
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
                
                // 找不到跑异常
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                // 根标签名字
                final String name = parser.getName();

                // 从打印信息可知是根标签
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }

                // 处理 merge 标签
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // 1，为 根标签 创建 根标签View
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    // 获取布局参数
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // 2，开始解析 根标签View下 下的 子view
                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // 将 根标签View 作为 子View 添加到入参的 root
                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                    // 若入参的 root 为空，则将 根标签View 直接充当 root
                    // 例如：PhoneWinow#generateLayout -> DecorView#onResourcesLoaded 函数中：
                    // inflater.inflate(layoutResource, null)
                }

            } catch (XmlPullParserException e) {
                ...
            } finally {
                ...
            }

            return result;
            // 返回 入参root，若 入参root 为空，result 也是根标签View
            // 否则 入参root 就是 根标签View 的 父View
        }
    }
```

<br>
注释1，创建 根标签View；<br>
注释2，解析 根标签View下 的 子view，为其创建 view 实例并添加到父容器中；<br>

先看注释1，
goin `createViewFromTag(..)`

##### createViewFromTag 函数源码

```
// LayoutInflater.java

    private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
        return createViewFromTag(parent, name, context, attrs, false);
    }

    View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        // Apply a theme wrapper, if allowed and one is specified.
        if (!ignoreThemeAttr) {
            final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
            final int themeResId = ta.getResourceId(0, 0);
            if (themeResId != 0) {
                context = new ContextThemeWrapper(context, themeResId);
            }
            ta.recycle();
        }

        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

        try {
            View view;
            // 1，优先 mFactory2 创建 view
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                // 2，其次用 mFactory 创建 view
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                // 3，然后用 mPrivateFactory 创建 view
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    // 4，最后调用自身 onCreateView 函数创建 view
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } catch (InflateException e) {
            ...
        } catch (ClassNotFoundException e) {
            ...
        } catch (Exception e) {
            ...
        }
    }
```

<br>
从注释1到4，我们看到了创建 view 方式的优先顺序，更详细源码，可以参阅 [LayoutInflater.java 源码](https://www.androidos.net.cn/androidossearch?query=LayoutInflater&sid=&from=code)；

此处仅看注释4的 onCreateView 函数，goin `view = onCreateView(parent, name, attrs)`

##### onCreateView 函数源码

```
// LayoutInflater.java

    protected View onCreateView(View parent, String name, AttributeSet attrs)
            throws ClassNotFoundException {
        // 同名函数
        return onCreateView(name, attrs);
    }
    
    protected View onCreateView(String name, AttributeSet attrs)
            throws ClassNotFoundException {
        return createView(name, "android.view.", attrs);
    }
    
    // 真正做事的函数
    public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        if (constructor != null && !verifyClassLoader(constructor)) {
            constructor = null;
            sConstructorMap.remove(name);
        }
        Class<? extends View> clazz = null;

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

            if (constructor == null) {
                // 1，用类加载器加载给 view 对应的类型
                // Class not found in the cache, see if it's real, and try to add it
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);

                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                
                // 缓存该类型的 constructor
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {
                    // 2，constructor 存在，但该类没加载过，也得加载类
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) : name).asSubclass(View.class);

                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
            }

            Object lastContext = mConstructorArgs[0];
            if (mConstructorArgs[0] == null) {
                // Fill in the context if not already within inflation.
                mConstructorArgs[0] = mContext;
            }
            Object[] args = mConstructorArgs;
            args[1] = attrs;

            // 3，用 constructor 创建一个 view 实例
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            mConstructorArgs[0] = lastContext;
            return view;

        } catch (NoSuchMethodException e) {
            ...
        } catch (ClassCastException e) {
            ...
        } catch (ClassNotFoundException e) {
            ...
        } catch (Exception e) {
            ...
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

<br>
结合注释1和注释2，该view 对应的类和其 constructor ，如果不存在，则需要加载，并缓存起来；<br>
注释3，通过 constructor 创建该view 的 View 实例，最终返回；

<br>
接着回到 inflate 函数源码的注释2，解析 根标签View下 的 子view <br>
goin `rInflateChildren(..)`

##### rInflateChildren 函数源码

```
// LayoutInflater.java

    final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        // 调用同名函数
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }
    
    // 真正做事的函数
    void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;
        boolean pendingRequestFocus = false;

        // 遍历直到结束标签为止
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            // 不是起始标签，就不处理
            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();

            if (TAG_REQUEST_FOCUS.equals(name)) {
                pendingRequestFocus = true;
                consumeChildElements(parser);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else { //1， 普通标签走这里
                // 创建 view 实例
                final View view = createViewFromTag(parent, name, context, attrs);
                // 获取父容器
                final ViewGroup viewGroup = (ViewGroup) parent; // 第二个入参
                // 获取参数
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                // 迭代的去加载 该View 的 子view
                rInflateChildren(parser, view, attrs, true);
                // 最后添加自身到第二个入参的父容器中
                viewGroup.addView(view, params);
            }
        }

        if (pendingRequestFocus) {
            parent.restoreDefaultFocus();
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
```

<br>
该函数的主要逻辑在注释1，每解析一个 `XmlPullParser.START_TAG` 的开始，都为其创建一个 View 实例，获取其父容器和参数，并迭代的去处理其 子view，最后把 该view 和其参数添加到其父容器中；<br>
此处创建 View 实例的 函数 createViewFromTag，跟 inflate 函数中注释1 处创建根标签 view 的是相同的；

<br>
##### 小结

由上述源码分析可知，PhoneWindow#setContentView 函数中的调用 `mLayoutInflater.inflate(layoutResID, mContentParent)`，<br>
真正会调用到<br>
 `inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) `，<br>
 该函数内，先为布局文件 R.layout.xxx 的 根标签 创建 View 实例，然后再调用 rInflateChildren 函数去处理 根标签View 的子view，函数内部有通过迭代的方式去处理 子view 的 子view，最终为其每个 view 都创建View 实例，并添加到其相应的父容器中；创建View 实例都是通过 createViewFromTag 函数实现的；
        
当 `mLayoutInflater.inflate(layoutResID, mContentParent)` 这句语句执行完后，<br>
Activity 的 onCreate 函数中 setContentView(R.layout.activity_xxx) 的布局文件 activity_xxx.xml，其 根view 被作为 子view 加载到  PhoneWindow.java 的 mContentParent 这个 id 为 content FrameLayout 类的 ViewGroup 中；<br>

此时 Activity 的视图树结构如下：

<br>
Activity 的视图树结构如下：

- （1层）DecorView
    - （2层）LinearLayout
        - （3层）ViewStub（id = action_mode_bar_stub ）
        - （3层）FrameLayout（ screen_title .xml 才有）
            - （4层）TextView（ screen_title .xml 才有，展示标题）
        - （3层）FrameLayout（id 为 content，即 mContentParent，装载 layout.xxx.xml 的父容器）
            - （4层）LinearLayout（activity_xxx.xml 的根view）
                - （5层）...


<br>
## 结尾

Activity 的 onCreate 函数，调用了 PhoneWindow 的 setContentView 函数，

之后的内容，简单的说就两步：
- 先加载 R.layout.screen_simple 到 mDecor；
- 再加载 R.layout.activity_xxx 到 mContentParent；

<br>
**setContentView 的工作主要概括如下**：

* 第4步 **installDecor()** 
  * generateDecor 函数
     * mDecor = new DecorView()

  * generateLayout 函数
     * layoutResource = R.layout.screen_simple 

     * mDecor.onResourcesLoaded(mLayoutInflater, layoutResource)
         * root = inflater.inflate(layoutResource, null)

     * mContentParent = findViewById(ID_ANDROID_CONTENT)

  * 执行结束后，Activity 的视图树构已经有三层结构


* 第5步 **mLayoutInflater.inflate(...)** 
  * temp = createViewFromTag(root, name, inflaterContext, attrs) // 创建 根view

      * view = onCreateView(parent, name, attrs)
      
  * rInflateChildren 函数
      * view = createViewFromTag(parent, name, context, attrs) // 创建 子view

      * rInflateChildren(parser, view, attrs, true) // 迭代

      * viewGroup.addView(view, params) // 添加到父容器

  * 执行结束后，为 Activity 的视图树构成了至少四层结构



## 拓展

### 查源码的链接

[查 Android 源码的链接](https://www.androidos.net.cn/androidossearch?query=&sid=&from=code)