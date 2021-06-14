---
layout:     post  
title:      "OkHttp 的第四个拦截器 ConnectInterceptor"  
subtitle:   "帮你建立连接"  
date:       2021-05-27 12:00:00  
author:     "GuoHao"  
header-img: "img/guitar.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - 三方库

---


## 前言

一句话概括 ConnectInterceptor 的作用：
> 创建一个连接，然后启动下一个拦截器，进行请求的发送和响应的接收。

工作模型描述：

为了连接复用，并没有直接创建连接，而是尝试从连接池找，找到能用的就直接用；
没找到再创建一个TCP连接，并放入连接池，以便复用；
如果一个TCP连接没有任务请求再使用，则超过一定时间就会被清除掉。

问题：

怎样的连接是能复用的？
- 相同 ip + port；
- 相同域名下的 不同 ip，相同 port，同时也是相同代理模式？？待求证。

## 正文

### 整体源码

```
// ConnectInterceptor.kt

/**
 * Opens a connection to the target server and proceeds to the next interceptor. The network might
 * be used for the returned response, or to validate a cached response with a conditional GET.
 */
 /**
 * 打开一个指向服务器的连接，然后执行下一个拦截器。
 * 网络可能被用来返回响应，或者用过附加的 GET 请求，鉴别一个已缓存的响应是否能用。
 */
object ConnectInterceptor : Interceptor {
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    // 前置工作，建立链接
    val exchange = realChain.call.initExchange(chain)
    val connectedChain = realChain.copy(exchange = exchange)
    // 中置工作，启动下一个 chain
    return connectedChain.proceed(realChain.request)
    // 无后置工作
  }
}

```

### RealCall.initExchange

初始化交换？就是 **建立连接** 啦！

```
// RealCall.kt

  /** Finds a new or pooled connection to carry a forthcoming request and response. */
  /** 找到一个新的或池中的连接去承载即将发生的请求和响应。 */
  internal fun initExchange(chain: RealInterceptorChain): Exchange {
    ...

    // 拿到一个编解码器
    // codec = coder & decoder，即编码和解码
    // http1 的请求，就用 http1 的 codec，不能用 http2 的 codec 
    val codec = exchangeFinder!!.find(client, chain)
    // 构建一个 Exchange 对象，这意味着连接建立了
    val result = Exchange(this, eventListener, exchangeFinder!!, codec)
    this.interceptorScopedExchange = result

    synchronized(connectionPool) {
      this.exchange = result
      this.exchangeRequestDone = false
      this.exchangeResponseDone = false
      return result // 返回建立好的连接，交给下一个拦截器使用。
    }
  }
 
 // 接下来的调用链：
 exchangeFinder!!.find(client, chain) ->
 findHealthyConnection ->
 findConnection ->
 // 第一次尝试拿连接，拿到就返回
 if (connectionPool.callAcquirePooledConnection(address, call, null, false))
 // 上边的没拿到，再一次，区别在于参数增加了一个 routes，这使得可拿到支持Http2合并的连接
 if (connectionPool.callAcquirePooledConnection(address, call, routes, false))
 
```

### ExchangeFinder.find

```
// ExchangeFinder.kt

  fun find(
    client: OkHttpClient,
    chain: RealInterceptorChain
  ): ExchangeCodec {
    try {
      // 查找健康的连接
      val resultConnection = findHealthyConnection(
          connectTimeout = chain.connectTimeoutMillis,
          readTimeout = chain.readTimeoutMillis,
          writeTimeout = chain.writeTimeoutMillis,
          pingIntervalMillis = client.pingIntervalMillis,
          connectionRetryEnabled = client.retryOnConnectionFailure,
          doExtensiveHealthChecks = chain.request.method != "GET"
      )
      return resultConnection.newCodec(client, chain)
    }...
  }
```

### ExchangeFinder.findHealthyConnection

```
  /**
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
   /**
   * 找到一个健康的连接并返回，如果找到的连接不健康，就重复直到找到为止
   */
  @Throws(IOException::class)
  private fun findHealthyConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    doExtensiveHealthChecks: Boolean
  ): RealConnection {
    while (true) { // 死循环
    
      // 查找到一位候选的连接
      val candidate = findConnection(
          connectTimeout = connectTimeout,
          readTimeout = readTimeout,
          writeTimeout = writeTimeout,
          pingIntervalMillis = pingIntervalMillis,
          connectionRetryEnabled = connectionRetryEnabled
      )

      // 判断候选的连接是否健康，不健康则继续循环
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        candidate.noNewExchanges()
        continue
      }

      // 健康则返回
      return candidate
    }
  }
```

### ExchangeFinder.findConnection

这是一个大函数！也是查找连接的主要逻辑函数。
我计划给函数分成四部分，分别解析。

```
// ExchangeFinder.kt

  /**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  @Throws(IOException::class)
  private fun findConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean
  ): RealConnection {
    var foundPooledConnection = false
    var result: RealConnection? = null
    var selectedRoute: Route? = null
    var releasedConnection: RealConnection?
    val toClose: Socket?
    
    // === 第一部分开始 ===
    synchronized(connectionPool) {
      if (call.isCanceled()) throw IOException("Canceled")

      val callConnection = call.connection // changes within this overall method
      releasedConnection = callConnection
      // toClose 是一个 Socket，是 call 持有的上一个连接的 Socket，
      // 如果 call 持有的连接不能在新建连接，即 noNewExchanges = true
      // 或者跟本次请求的地址不对，则要关闭的。
      toClose = if (callConnection != null && (callConnection.noNewExchanges ||
              !sameHostAndPort(callConnection.route().address.url))) {
        call.releaseConnectionNoEvents()
      } else {
        null
      }

      if (call.connection != null) {
        // We had an already-allocated connection and it's good.
        // 已经分配了连接，赋值给结果字段
        result = call.connection
        // 不需要释放连接了
        releasedConnection = null
      }

      if (result == null) {
        // The connection hasn't had any problems for this call.
        refusedStreamCount = 0
        connectionShutdownCount = 0
        otherFailureCount = 0

        // Attempt to get a connection from the pool.
        // 尝试从池里找一个连接
        if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
          foundPooledConnection = true
          result = call.connection
        } else if (nextRouteToTry != null) {
          selectedRoute = nextRouteToTry
          nextRouteToTry = null
        }
      }
    }
    
    // 关闭上次的连接
    toClose?.closeQuietly()

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection!!)
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result!!)
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result!!
      // 返回连接，这个连接，可能是从连接池里找到的，也可能是 call 对象内的连接还能用
      // 只要可以建立连接，且地址相同，就能继续用
    }
    // === 第一部分结束 ===

    // === 第二部分开始 ===
    // 找不着，则需要第二次，尝试找同一个 Selection 下的不同 route
    // If we need a route selection, make one. This is a blocking operation.
    var newRouteSelection = false
    if (selectedRoute == null && (routeSelection == null || !routeSelection!!.hasNext())) {
      var localRouteSelector = routeSelector
      if (localRouteSelector == null) {
        localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
        this.routeSelector = localRouteSelector
      }
      newRouteSelection = true
      routeSelection = localRouteSelector.next()
    }

    var routes: List<Route>? = null
    synchronized(connectionPool) {
      if (call.isCanceled()) throw IOException("Canceled")

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        // 得到的是同代理模式下的不同 ip，指向一个域名
        routes = routeSelection!!.routes
        if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
          foundPooledConnection = true
          result = call.connection
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection!!.next()
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        result = RealConnection(connectionPool, selectedRoute!!)
        connectingConnection = result
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result!!)
      return result!!
    }
    // === 第二部分结束 ===

    // === 第三部分开始 ===
    // Do TCP + TLS handshakes. This is a blocking operation.
    result!!.connect(
        connectTimeout,
        readTimeout,
        writeTimeout,
        pingIntervalMillis,
        connectionRetryEnabled,
        call,
        eventListener
    )
    call.client.routeDatabase.connected(result!!.route())

    var socket: Socket? = null
    synchronized(connectionPool) {
      connectingConnection = null
      // Last attempt at connection coalescing, which only occurs if we attempted multiple
      // concurrent connections to the same host.
      // 创建连接后再尝试查找一次
      if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
        // We lost the race! Close the connection we created and return the pooled connection.
        // 找到了还是用池中的连接，关闭新建的连接
        result!!.noNewExchanges = true
        socket = result!!.socket()
        result = call.connection

        // It's possible for us to obtain a coalesced connection that is immediately unhealthy. In
        // that case we will retry the route we just successfully connected with.
        nextRouteToTry = selectedRoute
      } else {
        connectionPool.put(result!!)
        call.acquireConnectionNoEvents(result!!)
      }
    }
    socket?.closeQuietly()

    eventListener.connectionAcquired(call, result!!)
    return result!!
    // === 第三部分结束 ===
  }
```


### 第一次，从池中拿连接

在 ExchangeFinder.findConnection 函数的第一部分中：

```
connectionPool.callAcquirePooledConnection(address, call, null, false)
```

#### RealConnectionPool.callAcquirePooledConnection

```
// RealConnectionPool.kt

  /**
   * Attempts to acquire a recycled connection to [address] for [call]. Returns true if a connection
   * was acquired.
   *
   * If [routes] is non-null these are the resolved routes (ie. IP addresses) for the connection.
   * This is used to coalesce related domains to the same HTTP/2 connection, such as `square.com`
   * and `square.ca`.
   */
   // 
  fun callAcquirePooledConnection(
    address: Address,
    call: RealCall,
    routes: List<Route>?,
    requireMultiplexed: Boolean
  ): Boolean {
    this.assertThreadHoldsLock()
    // 遍历连接池
    for (connection in connections) {
      if (requireMultiplexed && !connection.isMultiplexed) continue // 要求多路复用而该连接不是，则循环下一个
      if (!connection.isEligible(address, routes)) continue // 该连接不合适，下一个
      // call 对象的内部字段，引用这个连接
      call.acquireConnectionNoEvents(connection)
      return true
    }
    return false
  }
```

#### 参数1，address 解析

address 是 Address 类型。

address 参数什么时候创建的？
在第一个拦截器的前置工作中，追源码：

```
// RetryAndFollowUpInterceptor
call.enterNetworkInterceptorExchange(request, newExchangeFinder)

// RealCall.kt
  fun enterNetworkInterceptorExchange(request: Request, newExchangeFinder: Boolean) {
    check(interceptorScopedExchange == null)
    check(exchange == null) {
      "cannot make a new request because the previous response is still open: " +
          "please call response.close()"
    }

    if (newExchangeFinder) {
      this.exchangeFinder = ExchangeFinder(
          connectionPool,
          createAddress(request.url), // 看这里，追
          this,
          eventListener
      )
    }
  }
  
  // 创建一个地址
  private fun createAddress(url: HttpUrl): Address {
    var sslSocketFactory: SSLSocketFactory? = null
    var hostnameVerifier: HostnameVerifier? = null
    var certificatePinner: CertificatePinner? = null
    if (url.isHttps) {
      sslSocketFactory = client.sslSocketFactory
      hostnameVerifier = client.hostnameVerifier
      certificatePinner = client.certificatePinner
    }

    return Address(
        uriHost = url.host,
        uriPort = url.port, // 这两个参数从 url 来
        dns = client.dns,  // 其他从 client 来
        socketFactory = client.socketFactory,
        sslSocketFactory = sslSocketFactory,
        hostnameVerifier = hostnameVerifier,
        certificatePinner = certificatePinner,
        proxyAuthenticator = client.proxyAuthenticator,
        proxy = client.proxy,
        protocols = client.protocols,
        connectionSpecs = client.connectionSpecs,
        proxySelector = client.proxySelector
    )
  }
```

一个 Address 对象，带有主机名、端口、dns、代理 proxy、协议等字段。

##### newExchangeFinder 字段

注意这个 newExchangeFinder 变量，它决定了是否 createAddress(...)；
如果是重试，则应该不创建，这是可以看看第一个拦截器 RetryAndFollowUpInterceptor 的 intercept 函数中的后置工作：

```
catch (e: RouteException) { // 链路异常
                // The attempt to connect via a route failed. The request will not have been sent.
                if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
                    throw e.firstConnectException
                }
                // 执行到这，没抛异常，能重试，则不用新的 ExchangeFinder
                newExchangeFinder = false
                // 下一次循环
                continue
            } catch (e: IOException) { // IO 超时等错误
                // An attempt to communicate with a server failed. The request may have been sent.
                if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
                    throw e
                }
                // 执行到这，没抛异常，能重试，则不用新的 ExchangeFinder
                newExchangeFinder = false
                // 下一次循环
                continue 
            }
```

可见，出错后如果还能重试，newExchangeFinder = false，为的就是重试的时候，ConnectInterceptor 的 intercept 函数不要新建地址，即不要执行 createAddress(...)；

#### RealConnection.isEligible

重点看看连接是否合适！
对于第一次查找连接的时候，主要看两点：
- 链接数没超过限制
- 通过同一方式，连接到同一个主机

```
// RealConnection.kt

  internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
    // If this connection is not accepting new exchanges, we're done.
    if (calls.size >= allocationLimit || noNewExchanges) return false
    // 比较该 connection 的承载数
    // 插曲：http2 之前，每一个 TCP 链接，同一时间只能承载一个请求
    // allocationLimit 默认为 1

    // If the non-host fields of the address don't overlap, we're done.
    if (!this.route.address.equalsNonHost(address)) return false
    // 里面比较了 10 项：
    // dns、proxyAuthenticator、protocols、connectionSpecs、
    // proxySelector、proxy、sslSocketFactory、
    // hostnameVerifier、certificatePinner、url.port

    // 单独比较域名 host
    // If the host exactly matches, we're done: this connection can carry the address.
    if (address.url.host == this.route().address.url.host) {
      return true // This connection is a perfect match.
    }
    
    // ip 一样但域名不一样，可能是虚拟主机；
    // 没关系，继续往下走...

    // 下面是 http2 的比较
    // At this point we don't have a hostname match. But we still be able to carry the request if
    // our connection coalescing requirements are met. 
    // 这个时候，域名不匹配。但是我们仍可以执行请求，只要满足链接聚合的要求。
    // See also:
    // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
    // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

    // 1. This connection must be HTTP/2.
    if (http2Connection == null) return false

    // 2. The routes must share an IP address.
    if (routes == null || !routeMatchesAny(routes)) return false

    // 3. This connection's server certificate's must cover the new host.
    if (address.hostnameVerifier !== OkHostnameVerifier) return false
    if (!supportsUrl(address.url)) return false

    // 4. Certificate pinning must match the host.
    try {
      address.certificatePinner!!.check(address.url.host, handshake()!!.peerCertificates)
    } catch (_: SSLPeerUnverifiedException) {
      return false
    }

    return true // The caller's address can be carried by this connection.
  }
```

##### supportsUrl 函数

```
   fun supportsUrl(url: HttpUrl): Boolean {
    val routeUrl = route.address.url

    if (url.port != routeUrl.port) {
      return false // Port mismatch.
    }

    if (url.host == routeUrl.host) {
      return true // Host match. The URL is supported.
    }

    // We have a host mismatch. But if the certificate matches, we're still good.
    // 域名不匹配，但是证书匹配，还是可以重用
    return !noCoalescedConnections && handshake != null && certificateSupportHost(url, handshake!!)
  }
```

##### RealCall.acquireConnectionNoEvents

```
// RealCall.kt

  fun acquireConnectionNoEvents(connection: RealConnection) {
    connectionPool.assertThreadHoldsLock()

    check(this.connection == null)
    this.connection = connection // 引用这个连接
    connection.calls.add(CallReference(this, callStackTrace))
  }
```

#### RealConnection 类

上述的 connection 是 RealConnection 类。该类的结构：

```
  /** The low-level TCP socket. */
  private var rawSocket: Socket? = null

  /**
   * The application layer socket. Either an [SSLSocket] layered over [rawSocket], or [rawSocket]
   * itself if this connection does not use SSL.
   */
  private var socket: Socket? = null
  private var handshake: Handshake? = null
  private var protocol: Protocol? = null
  private var http2Connection: Http2Connection? = null
  private var source: BufferedSource? = null
  private var sink: BufferedSink? = null
```

可知，RealConnection 类持有 rawSocket、socket 字段，持有连接；
source、sink 字段，可以发送和接收数据。

#### RealConnectionPool 类

```
// RealConnectionPool.kt

  private val connections = ArrayDeque<RealConnection>()
    
  fun put(connection: RealConnection) {
    this.assertThreadHoldsLock()

    connections.add(connection)
    cleanupQueue.schedule(cleanupTask)
  }
```

原来存储连接的数据结构是一个 ArrayDeque 双段队列。
put 操作处了把连接存到 ArrayDeque 中，还启动一个 cleanupTask 任务。
该任务会调用 cleanup 方法清除池中的连接，主要依据 keepAliveDurationNs 字段来判断。


### 第二次，从池中拿连接

在 ExchangeFinder.findConnection 函数的第二部分中：

源码：

```
connectionPool.callAcquirePooledConnection(address, call, routes, false)
```

区别第一次调用的是参数3，routes。

#### 参数3，routes | Route 解析

参数 routes 是一个 List，元素类型为 Route。Route 类的定义：

```
class Route(
  @get:JvmName("address") val address: Address,
  /**
   * Returns the [Proxy] of this route.
   *
   * **Warning:** This may disagree with [Address.proxy] when it is null. When
   * the address's proxy is null, the proxy selector is used.
   */
  @get:JvmName("proxy") val proxy: Proxy,
  @get:JvmName("socketAddress") val socketAddress: InetSocketAddress
)
```

Route 类，主要包含三个字段：
- address（Address类）
    - dns 解析前的地址，只有域名没有 ip
- proxy（Proxy类）
    - 表示代理模式：直练，http代理，SOCKS代理；用来帮助判断 Http2 的连接是否可重用；
- socketAddress（InetSocketAddress类）
    - **socketAddress 就是 dns 解析后的 ip 地址！**

##### routeSelection 实例分析

> 主机：hencoder.com
> 
> 直连：
> - 1.2.3.4:443
> - 5.6.7.8:443
> 
> 代理：hencoderproxy.com
> - 9.10.11.12:443
> - 13.14.15.16:443

1.2.3.4:443 、5.6.7.8:443，算是独立的 Route；
1.2.3.4:443 和 5.6.7.8:443 算一个 Selection1；
9.10.11.12:443 和 13.14.15.16:443 算 Selection2；
Selection1 + Selection2 算是一个 RouteSelector。

> RouteSelector > Selection( routeSelection ) > Route
> RouteSelector 本质就是一个 list
> Selection 是 RouteSelector 的内部类，
> **同一个 Selection，端口一样，代理模式一样（要么直连，要么代理），只有 IP 不一样，比如上面的 Selection1、Selection2**
> Route 表示一个可用的路由

结构：
> - RouteSelector
>     - Selection1（直联）
>         - Route1：1.2.3.4:443
>         - Route2：5.6.7.8:443
>     - Selection2（代理）
>         - Route3：5.6.7.8:443
>         - Route4：13.14.15.16:443

##### selectedRoute 怎么拿

selectedRoute 就是发起连接时，**选中的那条路由**，即要连接哪个主机。

selectedRoute 主要从 routeSelection 来拿；

routeSelection 来自 routeSelector；

routeSelector 为空时，会新建：
`RouteSelector(address, call.client.routeDatabase, call, eventListener)`

#### route selection 获取 | dns 解析

有了上述的 routeSelection 分析，就比较容易看懂这一块了。
dns 的查找时机，在第二次从连接池查找连接之前，即：

```
// ExchangeFinder.kt
// findConnection 方法的第二部分：

    // === 第二部分开始 ===
    // If we need a route selection, make one. This is a blocking operation.
    var newRouteSelection = false
    // selectedRoute (ip + addr)
    // routeSelection (多个 selectedRoute，同代理模式不同ip)
    // routeSelector (多个 routeSelection)
    if (selectedRoute == null && (routeSelection == null || !routeSelection!!.hasNext())) {
      var localRouteSelector = routeSelector
      if (localRouteSelector == null) {
        localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
        this.routeSelector = localRouteSelector
      }
      // 这里导致第二次去连接池查找
      newRouteSelection = true
      // 这里触发 dns 查找，即把域名转成 ip，可能是多个 ip
      routeSelection = localRouteSelector.next()
    }

    var routes: List<Route>? = null
    synchronized(connectionPool) {
      if (call.isCanceled()) throw IOException("Canceled")

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        // 在此尝试，由于聚合连接，这次可能会成功找到连可用接。
        // 这些 routes，是相同代理模式下的不同 ip
        routes = routeSelection!!.routes
        // 第二次查找连接池
        if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
          foundPooledConnection = true
          result = call.connection
        }
      }
    ...
    // === 第二部分结束 ===
```

为何要在第二次查找连接池前，解析 dns，为何不在第一次？

```
routeSelection = localRouteSelector.next()

// 涉及的调用链
// RouteSelector.kt
-> while (hasNextProxy()) :
    -> val proxy = nextProxy()
        -> resetNextInetSocketAddress(result)
          -> val addresses = address.dns.lookup(socketHost) 
        // 返回一个域名下的多个 IP 地址，addresses -> inetSocketAddresses
    -> for : inetSocketAddresses -> routes
    -> return Selection(routes)
```

##### RouteSelector.next

```
// RouteSelector.kt

  operator fun next(): Selection {
    if (!hasNext()) throw NoSuchElementException()

    // Compute the next set of routes to attempt.
    val routes = mutableListOf<Route>()
    while (hasNextProxy()) {
      // Postponed routes are always tried last. For example, if we have 2 proxies and all the
      // routes for proxy1 should be postponed, we'll move to proxy2. Only after we've exhausted
      // all the good routes will we attempt the postponed routes.
      // 追踪这行
      val proxy = nextProxy()
      // nextProxy() 方法会更新到 inetSocketAddresses 字段
      for (inetSocketAddress in inetSocketAddresses) {
        val route = Route(address, proxy, inetSocketAddress)
        if (routeDatabase.shouldPostpone(route)) {
          postponedRoutes += route
        } else {
          routes += route
        }
      }

      if (routes.isNotEmpty()) {
        break
      }
    }

    if (routes.isEmpty()) {
      // We've exhausted all Proxies so fallback to the postponed routes.
      routes += postponedRoutes
      postponedRoutes.clear()
    }

    return Selection(routes)
  }
```
RouteSelector 的 next() 方法，返回 Selection 对象；
一个 Selection 对象，会含有多个 Route 对象（一个 Route = Ip + Port）。

其中的 nextProxy() 方法，返回的是 Proxy 对象。但是在执行过程中，更新了 inetSocketAddresses 列表字段，然后遍历，从中拿到多个 address 构建多个 Route 对象，再组成 Selection 对象最后返回。

继续追踪 nextProxy() 方法：

##### RouteSelector.nextProxy

```
// RouteSelector.kt

  /** Returns the next proxy to try. May be PROXY.NO_PROXY but never null. */
  @Throws(IOException::class)
  private fun nextProxy(): Proxy {
    if (!hasNextProxy()) {
      throw SocketException(
          "No route to ${address.url.host}; exhausted proxy configurations: $proxies")
    }
    val result = proxies[nextProxyIndex++]
    resetNextInetSocketAddress(result) // 继续跟
    return result
  }

```

##### RouteSelector.resetNextInetSocketAddress

```
  // 下一个函数：
  /** Prepares the socket addresses to attempt for the current proxy or host. */
  @Throws(IOException::class)
  private fun resetNextInetSocketAddress(proxy: Proxy) {
    // Clear the addresses. Necessary if getAllByName() below throws!
    val mutableInetSocketAddresses = mutableListOf<InetSocketAddress>()
    inetSocketAddresses = mutableInetSocketAddresses

    ...

    if (proxy.type() == Proxy.Type.SOCKS) {
      mutableInetSocketAddresses += InetSocketAddress.createUnresolved(socketHost, socketPort)
    } else {
      eventListener.dnsStart(call, socketHost)

      // Try each address for best behavior in mixed IPv4/IPv6 environments.
      // 在这里，dns 查找！返回 List<InetAddress>，即多个 ip。
      val addresses = address.dns.lookup(socketHost)
      if (addresses.isEmpty()) {
        throw UnknownHostException("${address.dns} returned no addresses for $socketHost")
      }

      eventListener.dnsEnd(call, socketHost, addresses)

      for (inetAddress in addresses) {
        mutableInetSocketAddresses += InetSocketAddress(inetAddress, socketPort)
      }
    }
  }
  // dns 查找到多个 ip，追踪存到全局变量的 inetSocketAddresses 字段中，一个 List。
  // 在上述的 RouteSelector 的 next() 方法中，inetSocketAddresses 会被遍历，
  // 最后得到多个 Route 对象，都成一个 Selection 对象，next 方法结束返回 Selection 对象。
```

可知：获取 routeSelection 时，就会触发 dns 解析，得到的多个 IP，对应多个 route，即 routes；
然后包裹在 Selection(routes) 中，返回；
然后赋值给 routeSelection；
现在可以解释 routeSelection 的意义了：
> **，它包含一个或多个 route，即一个域名下的多个 IP；
> 在后续去连接池中找可用连接时，就会根据 `routeSelection!!.routes` 去匹配！**

dns.lookup 方法，内部实现是依赖 InetAddress 类：

```
override fun lookup(hostname: String): List<InetAddress> {
        try {
          return InetAddress.getAllByName(hostname).toList()
        } ...
      }

// 继续追
// InetAddress.java

    public static InetAddress[] getAllByName(String host)
        throws UnknownHostException {
        // Android-changed: Resolves a hostname using Libcore.os.
        // Also, returns both the Inet4 and Inet6 loopback for null/empty host
        return impl.lookupAllHostAddr(host, NETID_UNSET).clone();
    }

// impl 是 Inet6AddressImpl 类
// Inet6AddressImpl.java

    @Override
    public InetAddress[] lookupAllHostAddr(String host, int netId) throws UnknownHostException {
        if (host == null || host.isEmpty()) {
            // Android-changed: Return both the Inet4 and Inet6 loopback addresses
            // when host == null or empty.
            return loopbackAddresses();
        }

        // Is it a numeric address?
        InetAddress result = InetAddressUtils.parseNumericAddressNoThrowStripOptionalBrackets(host);
        if (result != null) {
            return new InetAddress[] { result };
        }

        return lookupHostByName(host, netId);
    }

    private static InetAddress[] lookupHostByName(String host, int netId)
            throws UnknownHostException {
        ...
        
        try {
            ...            
            // 在这里，委托了 Libcore 去解析 dns，调用到了底层函数
            InetAddress[] addresses = Libcore.os.android_getaddrinfo(host, hints, netId);
            ...
            // 缓存
            addressCache.put(host, netId, addresses);
            // 返回查到的多个地址
            return addresses;
        }...
    }
```

可知，dns 解析最终是委托 Libcore 去解析的，没有在 Java 层用到 socket。

#### 再看 RealConnection.isEligible 

跟第一次从连接池找的区别，参数 routes。

代码片段：

```
// RealConnection.kt

  internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
  
    // 下面是 http2 的比较
    // At this point we don't have a hostname match. But we still be able to carry the request if
    // our connection coalescing requirements are met. 
    // 这个时候，域名不匹配。但是我们仍可以执行请求，只要满足链接聚合的要求。
    // See also:
    // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
    // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

    // 1. This connection must be HTTP/2.
    if (http2Connection == null) return false

    // 2. The routes must share an IP address.
    if (routes == null || !routeMatchesAny(routes)) return false
    // 没有 routes ，没法判断连接合并
```

### 第三次，拿不到连接，直接创建

在 ExchangeFinder.findConnection 函数的第三部分中：

相关源码：
```
// okhttp3/internal/connection/ExchangeFinder.kt
// findConnection 函数：

    // === 第三部分开始 ===

    // Do TCP + TLS handshakes. This is a blocking operation.
    result!!.connect(
        connectTimeout,
        readTimeout,
        writeTimeout,
        pingIntervalMillis,
        connectionRetryEnabled,
        call,
        eventListener
    )
    call.client.routeDatabase.connected(result!!.route())

```

##### 第三次，从池中拿连接


```
// ExchangeFinder.kt
// findConnection 方法的最后一部分：

    var socket: Socket? = null
    synchronized(connectionPool) {
      connectingConnection = null
      // Last attempt at connection coalescing, which only occurs if we attempted multiple
      // concurrent connections to the same host.
      // 最后一次尝试找合并连接，只会在尝试了对相同主机的多路复用连接之后
      if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
        // We lost the race! Close the connection we created and return the pooled connection.
        // 找到池中的连接，则优先使用，关闭刚刚创建的连接
        result!!.noNewExchanges = true
        socket = result!!.socket()
        result = call.connection

        // It's possible for us to obtain a coalesced connection that is immediately unhealthy. In
        // that case we will retry the route we just successfully connected with.
        // 这有一种可能就是我们获取到的聚合连接之后，该连接立刻不可用。
        // 这种情况下，我们应该尝试重试刚刚连接的路由。
        nextRouteToTry = selectedRoute
      } else {
        // 走到这，表示池中没有可以连接，采用新建的连接并放入池中
        connectionPool.put(result!!)
        call.acquireConnectionNoEvents(result!!)
      }
    }
    socket?.closeQuietly()

    eventListener.connectionAcquired(call, result!!)
    return result!!
    // === 第三部分结束 ===
```

##### 极端情况

这是考虑到一种极端情况，比如：

> 两个并发的连接，同时访问：
> - hencoder.com/1
> - hencoder.com/2

> 两个并发的连接，都创建了 connection；
> 然后有一个先执行同步方法，即 第三次的 RealConnectionPool#callAcquirePooledConnection ；
> 如果没拿到连接，就把此次新建的连接放到线程池；
> 然后第二个执行同步方法，即 第三次的 RealConnectionPool#callAcquirePooledConnection ；
> 就会拿到池里的连接，然后把新建的连接丢掉；
> 但是，会做这么一件事：`nextRouteToTry = selectedRoute`
> ？？？这个 selectedRoute 在上面是用来新建连接的：
> `result = RealConnection(connectionPool, selectedRoute!!)`
> 然后发起连接：`result!!.connect`

#### 创建连接的过程

核心代码：

```
// RealConnection.kt

  fun connect(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    call: Call,
    eventListener: EventListener
  ) {
  ...
   while (true) {
      try {
        if (route.requiresTunnel()) {
          // 关键点一
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener)
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break
          }
        } else {
          // 关键点二
          connectSocket(connectTimeout, readTimeout, call, eventListener)
        }
        // 关键点三
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
        eventListener.connectEnd(call, route.socketAddress, route.proxy, protocol)
        break
      }catch (e: IOException) {
        ...
      }
  
  
  }
```

##### 关键点一

connectTunnel 是怎么回事？
这是一种 http 代理 https 的模式。
追踪调用链：
```
// RealConnection.kt

  private fun connectTunnel(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    call: Call,
    eventListener: EventListener
  ) {
    var tunnelRequest: Request = createTunnelRequest()
    val url = tunnelRequest.url
    for (i in 0 until MAX_TUNNEL_ATTEMPTS) {
      // 也是先常规的连接
      // 建立连接后，会拿到 source、sink 两个字段，前者收信息，后置发信息
      connectSocket(connectTimeout, readTimeout, call, eventListener)
      // 创建通道
      tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url)
          ?: break // Tunnel successfully created.
      // 这是很标准的 Tunnel
      // 会额外发送一些 Tunnel 相关的信息，通过上述连接的 sink 字段
      // 比如：val requestLine = "CONNECT ${url.toHostHeader(includeDefaultPort = true)} HTTP/1.1"
        ...
    }
  }
```

##### 关键点二

接着看关键点二：
```
// RealConnection.kt

  /** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
  @Throws(IOException::class)
  private fun connectSocket(
    connectTimeout: Int,
    readTimeout: Int,
    call: Call,
    eventListener: EventListener
  ) {
    val proxy = route.proxy
    val address = route.address

    val rawSocket = when (proxy.type()) {
      Proxy.Type.DIRECT, Proxy.Type.HTTP -> address.socketFactory.createSocket()!!
      else -> Socket(proxy)
    }
    this.rawSocket = rawSocket

    eventListener.connectStart(call, route.socketAddress, proxy)
    rawSocket.soTimeout = readTimeout
    try {
      Platform.get().connectSocket(rawSocket, route.socketAddress, connectTimeout)
    } catch (e: ConnectException) {
      throw ConnectException("Failed to connect to ${route.socketAddress}").apply {
        initCause(e)
      }
    }

    // The following try/catch block is a pseudo hacky way to get around a crash on Android 7.0
    // More details:
    // https://github.com/square/okhttp/issues/3245
    // https://android-review.googlesource.com/#/c/271775/
    try {
      source = rawSocket.source().buffer()
      sink = rawSocket.sink().buffer()
    } catch (npe: NullPointerException) {
      if (npe.message == NPE_THROW_WITH_NULL) {
        throw IOException(npe)
      }
    }
  }
```

有 socketFactory ，就用它建 Socket；否则就直接新建：`Socket(proxy)` ；
**建立连接成功，即表示三次握手成功；**
建立连接后，会拿到 source、sink 两个字段，前者收信息，后置发信息。
值得注意，这里给 Socket 增加了拓展函数：
```
    source = rawSocket.source().buffer()
    sink = rawSocket.sink().buffer()
```
```
// jvmMain/okio/Okio.kt

fun Socket.source(): Source {
  val timeout = SocketAsyncTimeout(this)
  val source = InputStreamSource(getInputStream(), timeout)
  return timeout.source(source)
}

fun Socket.sink(): Sink {
  val timeout = SocketAsyncTimeout(this)
  val sink = OutputStreamSink(getOutputStream(), timeout)
  return timeout.sink(sink)
}
```

可见 source 关联 InputStreamSource，即输入流；sink 关联 OutputStreamSink，即输出流；

##### 关键点三

在创建了连接后，来到第三个关键点，建立协议：
```
// okhttp3/internal/connection/RealConnection.kt

    establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
```

```
// okhttp3/internal/connection/RealConnection.kt

  private fun establishProtocol(
    connectionSpecSelector: ConnectionSpecSelector,
    pingIntervalMillis: Int,
    call: Call,
    eventListener: EventListener
  ) {
    // 不需要加密，即不是 https
    if (route.address.sslSocketFactory == null) {
      // http2
      if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) {
        socket = rawSocket
        protocol = Protocol.H2_PRIOR_KNOWLEDGE
        // 关键点一，先发一段信息，相当于招手的东西
        startHttp2(pingIntervalMillis)
        return
      }

      socket = rawSocket
      protocol = Protocol.HTTP_1_1 // http1 直接返回
      return
    }

    eventListener.secureConnectStart(call)
    // 关键点二，加密连接
    connectTls(connectionSpecSelector)
    eventListener.secureConnectEnd(call, handshake)

    if (protocol === Protocol.HTTP_2) {
      startHttp2(pingIntervalMillis)
    }
  }

```

在关键点一中，发一段信息，相关代码：
```
// okhttp3/internal/http2/Http2Writer.kt
sink.write(CONNECTION_PREFACE)

// okhttp3/internal/http2/Http2.kt
val CONNECTION_PREFACE = "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n".encodeUtf8()
```

在关键点二中，建立加密连接：
```
  private fun connectTls(connectionSpecSelector: ConnectionSpecSelector) {
    val address = route.address
    val sslSocketFactory = address.sslSocketFactory
    var success = false
    var sslSocket: SSLSocket? = null
    try {
      // 创建一个 socket 的包裹
      // Create the wrapper over the connected socket.
      sslSocket = sslSocketFactory!!.createSocket(
          rawSocket, address.url.host, address.url.port, true /* autoClose */) as SSLSocket

      // Configure the socket's ciphers, TLS versions, and extensions.
      val connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket)
      if (connectionSpec.supportsTlsExtensions) {
        Platform.get().configureTlsExtensions(sslSocket, address.url.host, address.protocols)
      }

      // Force handshake. This can throw!
      sslSocket.startHandshake()
      // block for session establishment
      val sslSocketSession = sslSocket.session
      val unverifiedHandshake = sslSocketSession.handshake()

      // 开始验证证书
      // Verify that the socket's certificates are acceptable for the target host.
      if (!address.hostnameVerifier!!.verify(address.url.host, sslSocketSession)) {
        val peerCertificates = unverifiedHandshake.peerCertificates
        if (peerCertificates.isNotEmpty()) {
          val cert = peerCertificates[0] as X509Certificate
          throw SSLPeerUnverifiedException("""
              |Hostname ${address.url.host} not verified:
              |    certificate: ${CertificatePinner.pin(cert)}
              |    DN: ${cert.subjectDN.name}
              |    subjectAltNames: ${OkHostnameVerifier.allSubjectAltNames(cert)}
              """.trimMargin())
        } else {
          throw SSLPeerUnverifiedException(
              "Hostname ${address.url.host} not verified (no certificates)")
        }
      }

      val certificatePinner = address.certificatePinner!!

      handshake = Handshake(unverifiedHandshake.tlsVersion, unverifiedHandshake.cipherSuite,
          unverifiedHandshake.localCertificates) {
        certificatePinner.certificateChainCleaner!!.clean(unverifiedHandshake.peerCertificates,
            address.url.host)
      }

      // Check that the certificate pinner is satisfied by the certificates presented.
      certificatePinner.check(address.url.host) {
        handshake!!.peerCertificates.map { it as X509Certificate }
      }
      // 证书验证完毕

      // Success! Save the handshake and the ALPN protocol.
      val maybeProtocol = if (connectionSpec.supportsTlsExtensions) {
        Platform.get().getSelectedProtocol(sslSocket)
      } else {
        null
      }
      socket = sslSocket
      source = sslSocket.source().buffer()
      sink = sslSocket.sink().buffer()
      protocol = if (maybeProtocol != null) Protocol.get(maybeProtocol) else Protocol.HTTP_1_1
      success = true
    } finally {
      ...
    }
  }
```

### 回头看

#### Codec 类

再回头看这段代码

```
// ExchangeFinder.kt

  fun find(
    client: OkHttpClient,
    chain: RealInterceptorChain
  ): ExchangeCodec {
    try {
      // 返回一个可用的健康的连接
      val resultConnection = findHealthyConnection(
          connectTimeout = chain.connectTimeoutMillis,
          readTimeout = chain.readTimeoutMillis,
          writeTimeout = chain.writeTimeoutMillis,
          pingIntervalMillis = client.pingIntervalMillis,
          connectionRetryEnabled = client.retryOnConnectionFailure,
          doExtensiveHealthChecks = chain.request.method != "GET"
      )
      // 创建一个 Codec
      return resultConnection.newCodec(client, chain)
    } ...
  }
```

Codec 是什么，看源码：
```
// RealConnection.kt

  internal fun newCodec(client: OkHttpClient, chain: RealInterceptorChain): ExchangeCodec {
    val socket = this.socket!!
    val source = this.source!!
    val sink = this.sink!!
    val http2Connection = this.http2Connection

    return if (http2Connection != null) {
      Http2ExchangeCodec(client, this, chain, http2Connection)
    } else {
      socket.soTimeout = chain.readTimeoutMillis()
      source.timeout().timeout(chain.readTimeoutMillis.toLong(), MILLISECONDS)
      sink.timeout().timeout(chain.writeTimeoutMillis.toLong(), MILLISECONDS)
      Http1ExchangeCodec(client, this, source, sink)
    }
  }
```

Codec 是一个编码解码器，且 http1 和 http2 的编码解码是不同的，所以会分开创建，看源码：

##### source, sink

Http1ExchangeCodec 的构成函数的参数中，**source, sink，就是用来发送报文和接收报文的两个流**。sink 用来写，source 用来读。

```
// RealConnection.kt

  private var source: BufferedSource? = null
  private var sink: BufferedSink? = null
```

##### Http1ExchangeCodec

```
// Http1ExchangeCodec.kt 

  /** Returns bytes of a request header for sending on an HTTP transport. */
  fun writeRequest(headers: Headers, requestLine: String) {
    check(state == STATE_IDLE) { "state: $state" }
    // 请求行
    sink.writeUtf8(requestLine).writeUtf8("\r\n")
    // 报文头
    for (i in 0 until headers.size) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n")
    }
    // 报文头 和 报文体之间的换行
    sink.writeUtf8("\r\n")
    state = STATE_OPEN_REQUEST_BODY
  }
```

##### Http2ExchangeCodec

```
// Http2ExchangeCodec.kt

  override fun writeRequestHeaders(request: Request) {
    if (stream != null) return

    val hasRequestBody = request.body != null
    val requestHeaders = http2HeadersList(request)
    stream = http2Connection.newStream(requestHeaders, hasRequestBody)
    // We may have been asked to cancel while creating the new stream and sending the request
    // headers, but there was still no stream to close.
    if (canceled) {
      stream!!.closeLater(ErrorCode.CANCEL)
      throw IOException("Canceled")
    }
    stream!!.readTimeout().timeout(chain.readTimeoutMillis.toLong(), TimeUnit.MILLISECONDS)
    stream!!.writeTimeout().timeout(chain.writeTimeoutMillis.toLong(), TimeUnit.MILLISECONDS)
  }
```
可以看到，**http1 是用明文的方式来传信息的，而 http2 是用 stream 的形式**；

总结 Codec：**是一个编码器，带有http1或者http2的编码规则，带有健康连接的读写工具；**

#### Exchange 类

再看 RealCall.initExchange 函数：

```
// RealCall.kt

  internal fun initExchange(chain: RealInterceptorChain): Exchange {
    synchronized(connectionPool) {
      check(!noMoreExchanges) { "released" }
      check(exchange == null)
    }
    // 创建了解码器，
    // 现在只需要往它里面写东西，就可以发请求了
    // 从里面读东西，就能拿到服务器返回的响应了
    val codec = exchangeFinder!!.find(client, chain)
    // 创建 Exchange 对象，装入 codec
    // 如果说 codec 是一个读写工具，
    // 那么 Exchange 就是一个读写管理员，管理网络报文的交互
    val result = Exchange(this, eventListener, exchangeFinder!!, codec)
    this.interceptorScopedExchange = result

    synchronized(connectionPool) {
      this.exchange = result
      this.exchangeRequestDone = false
      this.exchangeResponseDone = false
      return result
    }
  }
```

Exchange 原意为交换，网络请求和响应，一来一回就是一种交换或交互；
所以，Exchange 就是一个**报文读写管理员，管理网络报文的交互**。

## 大结

没想到，这个拦截器挖了这么深。

明白了 Selector、Selection、Route 的结构关系（参考标题 “routeSelection 实例分析”）。
明白了 Route 类中的 Address、Proxy、InetSocketAddress 的关系。
明白了 RealConnection、RealConnectionPool 类的主要结构。

明白了 dns 查询会得到一个或多个 ip。

有了以上这些为基础，查找连接的过程就变得非常简单。

然后还明白了，codec 是一个读写工具，内部依赖 source 对象读数据，sink 对象写数据，它两是类似 Buffer 的存在
Exchange 就是一个读写管理员，管理网络报文的交互。内部封装一些批量、组合操作，诸如：writeRequestHeaders、createRequestBody、readResponseHeaders 等方法。
为后面的拦截器 CallServerInterceptor 的发送和接收信息做准备。