---
layout:     post  
title:      "Glide 探索 into 调用链"  
subtitle:   "先宏观把握整体"  
date:       2021-06-01 12:00:00  
author:     "GuoHao"  
header-img: "img/guitar.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - 三方库

---

## 前言

参考和引用：[Android主流三方库源码分析（三、深入理解Glide源码）](https://juejin.cn/post/6844904049595121672#heading-25)

Glide 版本：`api 'com.github.bumptech.glide:glide:4.12.0'`

本次文章尝试用调用链的角度分析，可以更快速的分析整个流程；
如果需要深入某个细节，再对照着去看源码，不会迷路；
同时可以个参考上述链接的源码解析和流程图等内容。

## 直观调用链

into 方法追起：

### into

> // RequestBuilder.java
> 
> into(@NonNull ImageView view)
> 
> > into(
> >         glideContext.buildImageViewTarget(view, transcodeClass),
> >         /*targetListener=*/ null,
> >         requestOptions,
> >         Executors.mainThreadExecutor());
> > 
> > > // 创建请求
> > > Request request = buildRequest(target);
> > > 
> > > > return buildRequestRecursive(.../* 传入更多的参数 */);
> > > > 
> > > > > // 此处创建：mainRequest、errorRequest、errorRequestCoordinator（异常处理对象，协调 mainRequest 与 errorRequest）
> > > > > 
> > > > > // 选择跟踪：mainRequest 的创建，递归建立缩略图请求
> > > > > Request mainRequest = buildThumbnailRequestRecursive(...)
> > > > > 
> > > > > > // 会创建 fullRequest（正常图）、thumbRequest（缩略图）、coordinator（协调前两者）
> > > > > > // 选择追踪：fullRequest 的创建
> > > > > > obtainRequest
> > > > > > 
> > > > > > > return SingleRequest.obtain(.../* 十六个参数 */);
> > > 
> > > // 执行请求，此时回到了 RequestBuilder.java
> > > requestManager.track(target, request);
> > > 
> > > > // RequestManager.java（该类管理和启动请求，会接收生命周期事件来启动暂停请求）
> > > > requestTracker.runRequest(request);
> > > > 
> > > > > // RequestTracker.java（该类跟踪、取消、重启那些处理中、已完成、失败的请求，线程不安全且必须在主线程调用）
> > > > > request.begin();
> > > > > 
> > > > > > // SingleRequest # begin()
> > > > > > //若通过 override API 指定 size，直接调用 onSizeReady(overrideWidth, overrideHeight)
> > > > > > //否则，先调用 target.getSize(this) 获取 size，然后回调 onSizeReady(...)
> > > > > > //最终都会走到： 
> > > > > > onSizeReady(...)，得到 targetView 的 size（宽和高）
> > > > > > > engine.load(...) // 启动引擎，有十余个参数
> > > > > > > // engine 是 Engine 类
> > > > > > > > Engine # load(...)
> > > > > > > > 
> > > > > > > > #### 构建图片的唯一标识 key
> > > > > > > > // 根据宽和高，scaleType 等信息构建出来的 key，寻找图片缓存时，就是依据 key 来匹配
> > > > > > > > 
> > > > > > > > EngineKey key = keyFactory.buildKey(...)
> > > > > > > > 
> > > > > > > > #### 用 key 查找图片缓存
> > > > > > > > 
> > > > > > > > // 从缓存加载
> > > > > > > > memoryResource = 
> > > > > > > > loadFromMemory(key, isMemoryCacheable, startTime);
> > > > > > > > > // 从活跃资源（正在使用）中加载
> > > > > > > > > active = loadFromActiveResources(key)
> > > > > > > > > > active = activeResources.get(key)
> > > > > > > > > > // 内部是弱引用的 HashMap：
> > > > > > > > > > // Map< Key, ResourceWeakReference > activeEngineResources = new HashMap<>()
> > > > > > > > > 
> > > > > > > > > // 从 Lru 缓存中加载
> > > > > > > > > cached = loadFromCache(key)
> > > > > > > > > > cached = getEngineResourceFromCache(key)
> > > > > > > > > > > cached = cache.remove(key);//获取并移除
> > > > > > > > > > > // cache 是 MemoryCache 接口，LruResourceCache 实现类，内部是 LinkedHashMap 存储
> > > > > > > > > > 
> > > > > > > > > > // 若找到，会从 Lru 移除并加入到活跃资源中
> > > > > > > > > > activeResources.activate(key, cached);
> > > > > > > > 
> > > > > > > > // 成功则直接回调 SingleRequest # onResourceReady 方法，
> > > > > > > > // 然后 return null，否则继续：
> > > > > > > > 
> > > > > > > > if (memoryResource == null)
> > > > > > > > return waitForExistingOrStartNewJob(...)
> > > > > > > > 
> > > > > > > > > // EngineJob 类简介：
> > > > > > > > > // 管理一次 load，添加、移除回调，并在完成的时候通知
> > > > > > > > > EngineJob<R> engineJob = engineJobFactory.build(key,...)
> > > > > > > > > 
> > > > > > > > > // DecodeJob 类简介：
> > > > > > > > > // 
> > > > > > > > > DecodeJob<R> decodeJob = decodeJobFactory.build(..,key,..)
> > > > > > > > > 
> > > > > > > > > engineJob.start(decodeJob);
> > > > > > > > > 
> > > > > > > > > > // EngineJob.java # start
> > > > > > > > > > 
> > > > > > > > > > #### 没找到则开线程做网络请求
> > > > > > > > > > 
> > > > > > > > > > GlideExecutor executor = decodeJob.willDecodeFromCache()
> > > > > > > > > > ? diskCacheExecutor : getActiveSourceExecutor();
> > > > > > > > > > 
> > > > > > > > > > // 根据不同的配置采用不同的线程池使用 
> > > > > > > > > > diskCacheExecutor/sourceUnlimitedExecutor
> > > > > > > > > > /animationExecutor/sourceExecutor
> > > > > > > > > > 来执行最终的解码任务decodeJob
> > > > > > > > > > 
> > > > > > > > > > executor.execute(decodeJob);

into 的函数调用，到此结束；
into 函数是在主线程执行的，它做的事情有：
首先会创建一个请求（带有十六个参数，包括 view、scaleType、width、height 等等）；
然后会尝试从内存缓存中加载缓存图片（因为内存访问速度很快，所以直接在主线程中执行）；
如果内存没有缓存，则开始尝试共网络加载，这时候会生成一个 decodeJob 对象放入线程池；

相应的任务（会有多种任务）已经放入适当的线程池（会有多种线程池）执行；
接下来将在 **子线程执行 decodeJob 的 run 方法**：

### 网络请求

> // DecodeJob.java # run() -> runWrapped()
> 
> switch (runReason) 
> 
> case INITIALIZE -> 
> stage = getNextStage(Stage.INITIALIZE);
> currentGenerator = getNextGenerator();
> runGenerators()
> 
> // 完整情况下，会异步依次生成这里的 ResourceCacheGenerator、DataCacheGenerator 和 **SourceGenerator** 对象，设置自身为其回调接口，并在之后执行其中的 startNext()
> 
> > currentGenerator.startNext()
> > 
> > > 网络请求是在 **SourceGenerator** 中发起的
> > > // SourceGenerator # startNext()
> > > while (!started && hasNextModelLoader()) {
> > > loadData = helper.getLoadData().get(loadDataListIndex++);
> > > if (loadData != null && ...) startNextLoad(loadData);
> > > > loadData.fetcher.loadData(...,new DataCallback() {...})
> > > > > HttpUrlFetcher # loadData(...) // Android 原生的网络加载组件
> > > > > 
> 
> case SWITCH_TO_SOURCE_SERVICE -> runGenerators()
> case DECODE_DATA -> decodeFromRetrievedData()

放入线程池的 decodeJob，其实就是为了做网络请求的。
当网络请求后，就要看看回调接口中做什么了。

### 网络回调

网络请求是在 SourceGenerator 中发起的，回调也是调用到 SourceGenerator 中。
当网络请求拿到相应数据，回调上述的 SourceGenerator 的 new DataCallback() 的接口：
> 在 SourceGenerator 类对象内：
> new DataCallback().onDataReady -> onDataReadyInternal(...)
> 
> // 网络回调后，从 SourceGenerator.java 开始：
> onDataReadyInternal(...)
> > cb.onDataFetcherReady(...) // cb 即 DecodeJob.java
> > 
> > // 来到 DecodeJob.java
> > onDataFetcherReady(loadData.sourceKey,data,...,originalKey)
> > > decodeFromRetrievedData()
> > > > 
> > > > // 核心代码：从数据中解码得到资源
> > > > resource = 
> > > > decodeFromData(currentFetcher, currentData, currentDataSource);
> > > > > 
> > > > > Resource<R> result = 
> > > > > decodeFromFetcher(data, dataSource);
> > > > > 
> > > > > > // 核心代码：将解码任务分发给LoadPath
> > > > > > return runLoadPath(data, dataSource, path);
> > > > > > 
> > > > > > return path.load(rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource));
> > > > > > 
> > > > > > > // LoadPath # load
> > > > > > > return loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
> > > > > > > 
> > > > > > > > // 核心代码：
> > > > > > > > for : path in decodePaths
> > > > > > > > result = path.decode(rewinder, width, height, options, decodeCallback); 
> > > > > > > > // 将解码任务又进一步分发给 DecodePath 的 decode 方法
> > > > > > > > 
> > > > > > > > // DecodePath # decode
> > > > > > > > 
> > > > > > > > > decodeResource()
> > > > > > > > > > decodeResourceWithList()
> > > > > > > > > > 
> > > > > > > > > > > for : decoder in decoders
> > > > > > > > > > > 
> > > > > > > > > > > // 判断该解码器是否有能力解码当前的数据 data
> > > > > > > > > > > if (decoder.handles(data, options))
> > > > > > > > > > > 
> > > > > > > > > > > > // 核心代码：分发给有能力处理的解码器 Decoder
> > > > > > > > > > > > result = decoder.decode(data, width, height, options);

这一段的意义在于找到一个合适的 **图片解码器**，去解码图片。

通过一连串调用，最终会执行到了
 `decoder.decode(data, width, height, options); ` 这行代码，
decode 是一个 ResourceDecoder <DataType, ResourceType> 接口（资源解码器），
根据不同的 DataType 和 ResourceType 它会有不同的实现类，这里的实现类是 **ByteBufferBitmapDecoder**，接着看其具体的解码流程。

怎么知道是 ByteBufferBitmapDecoder 呢？该其他博客将的，其实还有待自己求证。

#### 图片压缩解码

> DecodeJob # decodeFromRetrievedData
> 
> > resource = decodeFromData(currentFetcher, currentData, currentDataSource);
> >  
> > // 跳过中间调用，找到对应的解码器...
> > 
> > > // ByteBufferBitmapDecoder # decode
> > > 
> > > InputStream is = ByteBufferUtil.toStream(source);
> > > return downsampler.decode(is, width, height, options);
> > > 
> > > > // Downsampler # decode
> > > > 
> > > > return decode(is, outWidth, outHeight, options, EMPTY_CALLBACKS);
> > > > 
> > > > > return decode(new ImageReader.InputStreamImageReader(is, parsers, byteArrayPool), ...);
> > > > > 
> > > > > > Bitmap result = decodeFromWrappedStreams(...)
> > > > > > 
> > > > > > > // 省去计算压缩比例等一系列非核心逻辑
> > > > > > > 
> > > > > > > Bitmap downsampled = decodeStream(imageReader, options, callbacks, bitmapPool);
> > > > > > > 
> > > > > > > > result = imageReader.decodeBitmap(options);
> > > > > > > > 
> > > > > > > > > InputStreamImageReader # decodeBitmap
> > > > > > > > > 
> > > > > > > > > // 可知最终用到了 android.graphics 包的 
> > > > > > > > > // BitmapFactory 类的 decodeStream 方法
> > > > > > > > > return BitmapFactory.decodeStream(dataRewinder.rewindAndGet(), null, options);
> > > > > > > > > 
> > > > > > > > 
> > > > > > > 
> > > > > > > // 通知解码完成
> > > > > > > callbacks.onDecodeComplete(bitmapPool, downsampled);
> > > > > > 
> > > > > > // Bitmap 包装到 BitmapResource 的对象中返回
> > > > > > return BitmapResource.obtain(result, bitmapPool);
> > > > > > 
> > 
> > // 上述解码得到的资源不为空，则通知
> > if (resource != null) 
> > notifyEncodeAndRelease(resource, currentDataSource, isLoadingFromAlternateCacheKey);
> > 
> > > notifyComplete(result, dataSource, isLoadedFromAlternateCacheKey);
> > > 
> > > > callback.onResourceReady(resource, dataSource, isLoadedFromAlternateCacheKey);
> > > > // callback 是 EngineJob 类
> > > > 
> > > > > EngineJob # onResourceReady
> > > > > 
> > > > > > notifyCallbacksOfResult();
> > > > > > 
> > > > > > > engineJobListener.onEngineJobComplete(this, localKey, localResource);
> > > > > > > // engineJobListener 是 Engine 类
> > > > > > > Engine # onEngineJobComplete
> > > > > > > 
> > > > > > > > if (resource != null && resource.isMemoryCacheable())
> > > > > > > > activeResources.activate(key, resource);
> > > > > > > > // 这就是缓存！缓存到活跃资源，释放后则缓存到 Lru。
> > > > > > > 
> > > > > > > for : 
> > > > > > > entry.executor.execute(new CallResourceReady(entry.cb));
> > > > > > > // 这其实就是为了切换线程

到此可知，
解码时，就包含 **计算压缩比例** 这部分的工作，调用 Downsampler#decodeFromWrappedStreams 这个核心方法，进行压缩计算等工作；

解码完成后，会发起通知并通知到 EngineJob 类对象的 notifyCallbacksOfResult() 方法；该方法做了两件非常重要的事：
- 回调 Engine # onEngineJobComplete 进行图片缓存操作
- 委托 executor 进行线程切换，在主线程继续执行 CallResourceReady，更新 UI，显示图片

##### 图片压缩

在网络响应后，接着进行图片解码，解压的关键就是 **压缩比例计算**。

Downsampler#decodeFromWrappedStreams 方法是其中的核心方法。

#### 图片缓存

在网络响应后，图片压缩解码，解码完再进行缓存。

Engine # onEngineJobComplete

##### 如何转入到 Lru 缓存

此时的图片是缓存在活跃资源中的，若资源被释放，是如何转入到 Lru 缓存的？

#### 图片显示

图片压缩解码完，缓存完，会发出通知，最终会切换到主线程执行 更新UI操作：

> CallResourceReady # run
> > callCallbackOnResourceReady()
> > > cb.onResourceReady(engineResource, dataSource, isLoadedFromAlternateCacheKey);
> > > 
> > > SingleRequest # onResourceReady
> > > 
> > > > onResourceReady
> > > > 
> > > > > target.onResourceReady(result, animation);
> > > > > 
> > > > > ImageViewTarget # onResourceReady
> > > > > 
> > > > > > setResourceInternal(resource);
> > > > > > 
> > > > > > > setResource(resource);
> > > > > > > 
> > > > > > > > // 子类 BitmapImageViewTarget
> > > > > > > > view.setImageBitmap(resource);
> > > > > > > > 
> > > > > > > > // 子类 DrawableImageViewTarget
> > > > > > > > view.setImageDrawable(resource);
> > > > > > > > 
> > > > > > > > // 子类 ThumbnailImageViewTarget
> > > > > > > > // 缩略图，还需要调整宽高
> > > > > > > > ViewGroup.LayoutParams layoutParams = view.getLayoutParams();
> > > > > > > > Drawable result = getDrawable(resource);
> > > > > > > > result = new FixedSizeDrawable(result, layoutParams.width, layoutParams.height);
> > > > > > > > view.setImageDrawable(result);

到此，`Glide.with(Context).load("").into(View);` 的 into，
即 RequestBuilder 的 into 方法的调用链，基本最终完毕。

into 方法是 Glide 最复杂的调用，此次为了追踪调用链，省略了大量的细节。
单纯的调用链追踪，意义不大。
调用链追踪，是为了知道关键点在哪里。
是可以让我们在深入探索细节的时候，更加放心。
在追踪完了调用链后，对 Glide 的深入了解，才刚刚开始。

## 入口总结

网络请求完成，就会发起回调，然后走到下面的函数：

> ByteBufferBitmapDecoder # decode 方法，内部拿到 InputStream 流并解码。
> 再到 Downsampler 的 decodeFromWrappedStreams 方法。
> 该方法返回的 Bitmap，会被包装为 BitmapResource 对象。
> 然后发起回调 EngineJob # onResourceReady（带着 resource）
> > 再调用 EngineJob # notifyCallbacksOfResult();
> > > 然后发起回调 Engine # onEngineJobComplete 进行缓存处理
> > 
> > 然后 resource 放入 CallResourceReady 对象中，跟随该对象进入线程池。
> > 线程池切到主线程执行 CallResourceReady，最终会设置到一个 View 上。

### 图片网络请求前后的缓存在哪里看

当你像看图片缓存处理，从哪里开始看？

答：Engine # onEngineJobComplete(...)

网络请求前的查找缓存操作，在哪里看？

答：Engine # load(...)

### 图片解码、放缩在哪里看

当你像看图片解码、放缩等处理，从哪里开始看？

答：Downsampler 的 decodeFromWrappedStreams(...) 方法。

### 切线程在哪里看

EngineJob # notifyCallbacksOfResult();

## 总结

正如标题所示，通过调用链的跟中，找到了几个重要内容的关键函数。
可以放心的，专心的去研究了。


### 架构图

引用 [Android主流三方库源码分析（三、深入理解Glide源码）](https://juejin.cn/post/6844904049595121672#heading-25) 的图片：

![](/img/media/16195906155115/16234882892822.jpg)


### 流程图

引用 [Android主流三方库源码分析（三、深入理解Glide源码）](https://juejin.cn/post/6844904049595121672#heading-25) 的图片：


![](/img/media/16195906155115/16234158831821.jpg)
