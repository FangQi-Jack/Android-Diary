# **OkHttp 3.14.9**
当调用 Call 的 Enqueue 方法时，实现在 RealCall 中：
```java
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  transmitter.callStart();
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```
其中执行了 Dispather 的 enqueue 方法：
```java
  void enqueue(AsyncCall call) {
    synchronized (this) {
      readyAsyncCalls.add(call);

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      if (!call.get().forWebSocket) {
        AsyncCall existingCall = findExistingCallWithHost(call.host());
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
      }
    }
    promoteAndExecute();
  }
...
  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        asyncCall.callsPerHost().incrementAndGet();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
...
  @Nullable private AsyncCall findExistingCallWithHost(String host) {
    for (AsyncCall existingCall : runningAsyncCalls) {
      if (existingCall.host().equals(host)) return existingCall;
    }
    for (AsyncCall existingCall : readyAsyncCalls) {
      if (existingCall.host().equals(host)) return existingCall;
    }
    return null;
  }
```
在 enqueue 方法手机将当前 AsyncCall 放入 readyAsyncCalls 中，该队列用于保存准备好要执行的异步请求。如果当前请求不是 WebSocket 则会执行 findExistingCallWithHost 从已经存在的请求中查找与这个请求 host 相同的请求，如果存在则将新的 AsyncCall 的 callsPerHost 设置给已经存在的请求，该变量是 AtomicInteger 类型，用于记录 host 对应的真正运行的请求数。最后执行了 promoteAndExecute 方法，在 promoteAndExecute 方法中，通过遍历 readyAsyncCalls 找到所有可以立即执行的的 AsyncCall，并调用 AsyncCall 的 executeOn 方法：
```java
final class AsyncCall extends NamedRunnable {
    /**
     * Attempt to enqueue this async call on {@code executorService}. This will attempt to clean up
     * if the executor has been shut down by reporting the call as failed.
     */
    void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        executorService.execute(this);
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        transmitter.noMoreExchanges(ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
    }
    ...
        @Override protected void execute() {
      boolean signalledCallback = false;
      transmitter.timeoutEnter();
      try {
        Response response = getResponseWithInterceptorChain();
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } catch (Throwable t) {
        cancel();
        if (!signalledCallback) {
          IOException canceledException = new IOException("canceled due to " + t);
          canceledException.addSuppressed(t);
          responseCallback.onFailure(RealCall.this, canceledException);
        }
        throw t;
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
}
```
在 executeOn 方法中，将 AsynCall 添加到线程池中执行，AsyncCall 继承了 NameRunnable，在 NameRunnable 的 run 方法中会执行 AsyncCall 的 execute 方法，其中会执行 getResponseWithInterceptorChain 方法，该方法在 RealCall 中实现。
```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
      Response response = chain.proceed(originalRequest);
      if (transmitter.isCanceled()) {
        closeQuietly(response);
        throw new IOException("Canceled");
      }
      return response;
    } catch (IOException e) {
      calledNoMoreExchanges = true;
      throw transmitter.noMoreExchanges(e);
    } finally {
      if (!calledNoMoreExchanges) {
        transmitter.noMoreExchanges(null);
      }
    }
  }
```
在这里会获取创建 OkHttpClient 时添加的 Interceptor，然后创建 RetryAndFollowUpInterceptor、BridgeInterceptor、CacheInterceptor、ConnectInterceptor 以及 CallServerInterceptor，通过这些 Interceptor 创建 RealInterceptorChain，并执行它的 proceed 方法。
```java
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, transmitter, exchange);
  }

  public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
      throws IOException {
    ...
    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
        index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    ...
    return response;
  }
```
在 proceed 方法中会一次执行 Interceptor 的 intercept 方法。

## RetryAndFollowUpInterceptor
This interceptor recovers from failures and follows redirects as necessary.
这个拦截器主要用于失败重试和重定向。看它的 intercept 方法。
```java
@Override public Response intercept(Chain chain) throws IOException {
    // 获取当前请求
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    // 获取 Transmitter，该类用于连接应用和网络层
    Transmitter transmitter = realChain.transmitter();
    // 重定向次数
    int followUpCount = 0;
    // 上一次的请求结果
    Response priorResponse = null;
    while (true) {
        // 创建 ExchangeFinder
        transmitter.prepareToConnect(request);
    
      if (transmitter.isCanceled()) {
        throw new IOException("Canceled");
      }
    
      Response response;
      boolean success = false;
      try {
        // 执行下一个 Interceptor 的 intercept 方法
        response = realChain.proceed(request, transmitter, null);
        success = true;
      } catch (RouteException e) {
        // 路由异常，尝试重试，如果重试失败，则抛出第一个异常
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), transmitter, false, request)) {
          throw e.getFirstConnectException();
        }
        continue;
      } catch (IOException e) {
        // 尝试重新与服务器连接，如果重试失败则抛出异常
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, transmitter, requestSendStarted, request)) throw e;
        continue;
      } finally {
        // The network call threw an exception. Release any resources.
        if (!success) {
          transmitter.exchangeDoneDueToException();
        }
      }
    
      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }
    
    Exchange exchange = Internal.instance.exchange(response);
    Route route = exchange != null ? exchange.connection().route() : null;
    // 根据服务返回的响应，可能会添加认证头信息、重定向或处理连接超时。如果该请求无法继续被处理或出现的错误不需要继续处理，会返回null
    Request followUp = followUpRequest(response, route);
    
    // 无法重定向时直接返回 response
    if (followUp == null) {
        if (exchange != null && exchange.isDuplex()) {
          transmitter.timeoutEarlyExit();
        }
        return response;
    }
    
      RequestBody followUpBody = followUp.body();
      if (followUpBody != null && followUpBody.isOneShot()) {
        return response;
      }
    
      closeQuietly(response.body());
      if (transmitter.hasExchange()) {
        exchange.detachWithViolence();
      }
    // 如果超过最大重定向数则抛出异常
    if (++followUpCount > MAX_FOLLOW_UPS) {
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
    }
    
      request = followUp;
      priorResponse = response;
    }
}
```
在 RetryAndFollowUpInterceptor 的 intercept 方法中主要做了以下几件事：
1、通过 Transmitter 创建 ExchangeFinder，为连接做准备。
2、执行下一个 Interceptor 的 intercept 方法。
3、根据服务返回判断是否可以重定向。
4、如果可以重试则会复用上一次的连接或者新建连接进行重试，否则将服务器返回的响应数据包装后返回给用户。
5、判断重定向次数是否超过最大次数，是就抛出异常。

## BridgeInterceptor
这个拦截器的主要作用是桥接应用层和网络层，将用户请求转换成网络请求，然后请求网络，最后将网络响应转换成用户响应。
```java
  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
        // 交由 Okio 处理生成新的响应体
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }
```
BridgeInterceptor 主要完成以下工作：
1、从用户请求中创建 Request.Builder
2、添加网络请求头：Content-Type、Content-Length/Transfer-Encoding、Host、Connection、Accept-Encoding、Cookie、User-Agent
3、执行下一个 Interceptor 的 intercept
4、处理自定义 CookieJar，如果没有添加自定义 CookieJar 则无需处理
5、从网络响应中创建 Response.Builder
6、处理 Response.Builder，比如 Gzip 解压
7、返回网络响应

##  CacheInterceptor
CacheInterceptor 用于处理缓存相关逻辑。
```java
@Override public Response intercept(Chain chain) throws IOException {
    // 取出缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    
    long now = System.currentTimeMillis();
    // 获取缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    // 缓存中的请求
    Request networkRequest = strategy.networkRequest;
    // 缓存中的响应
    Response cacheResponse = strategy.cacheResponse;
    
    if (cache != null) {
        cache.trackResponse(strategy);
    }
    
    if (cacheCandidate != null && cacheResponse == null) {
        closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }
    
    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
        return new Response.Builder()
            .request(chain.request())
            .protocol(Protocol.HTTP_1_1)
            .code(504)
            .message("Unsatisfiable Request (only-if-cached)")
            .body(Util.EMPTY_RESPONSE)
            .sentRequestAtMillis(-1L)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();
    }
    
    // If we don't need the network, we're done.
    if (networkRequest == null) {
        return cacheResponse.newBuilder()
            .cacheResponse(stripBody(cacheResponse))
            .build();
    }
    
    Response networkResponse = null;
    try {
        // 执行下一个 Interceptor
        networkResponse = chain.proceed(networkRequest);
    } finally {
        // If we're crashing on I/O or otherwise, don't leak the cache body.
        if (networkResponse == null && cacheCandidate != null) {
            closeQuietly(cacheCandidate.body());
        }
    }
    
    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
        if (networkResponse.code() == HTTP_NOT_MODIFIED) {
            // 服务器返回 304，表示缓存有效，合并网络响应和缓存
            Response response = cacheResponse.newBuilder()
                .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build();
            networkResponse.body().close();
            
            // Update the cache after combining headers but before stripping the
            // Content-Encoding header (as performed by initContentStream()).
            cache.trackConditionalCacheHit();
            cache.update(cacheResponse, response);
            return response;
        } else {
            closeQuietly(cacheResponse.body());
        }
    }
    
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
    
    if (cache != null) {
        if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
            // Offer this request to the cache.
            CacheRequest cacheRequest = cache.put(response);
            return cacheWritingResponse(cacheRequest, response);
        }
        
        if (HttpMethod.invalidatesCache(networkRequest.method())) {
            try {
                cache.remove(networkRequest);
            } catch (IOException ignored) {
                // The cache cannot be written.
            }
        }
    }
    
    return response;
}
```
CacheInterceptor 的主要工作：
1、从缓存中获取数据，可能为 null
2、获取缓存策略
3、根据缓存策略做相应的处理

> 禁止使用网络和缓存，直接返回 504。
>
> 使用缓存，直接返回缓存数据。
>
> 同时使用网络和缓存，根据响应头判断使用哪一个，如果响应码为 304 则使用缓存并更新缓存使用网络响应，否则使用网络响应数据。

4、执行下一个 Interceptor
5、如果设置了缓存，则将数据更新到缓存中

##  ConnectInterceptor
用于连接服务器并执行下一个 Interceptor。
```java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Request request = realChain.request();
  Transmitter transmitter = realChain.transmitter();

  // We need the network to satisfy this request. Possibly for validating a conditional GET.
  boolean doExtensiveHealthChecks = !request.method().equals("GET");
  Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

  return realChain.proceed(request, transmitter, exchange);
}
```

ConnectInterceptor 的主要工作：

1、打开与服务器的网络连接。

2、执行下一个 Interceptor，即 CallServerInterceptor，由它来处理请求和获取数据。

ConnectInterceptor 的 intercept 方法中执行了 Transmitter 的 newExchange 方法：

```java
/** Returns a new exchange to carry a new request and response. */
Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
  synchronized (connectionPool) {
    if (noMoreExchanges) {
      throw new IllegalStateException("released");
    }
    if (exchange != null) {
      throw new IllegalStateException("cannot make a new request because the previous response "
          + "is still open: please call response.close()");
    }
  }

  ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);
  Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);

  synchronized (connectionPool) {
    this.exchange = result;
    this.exchangeRequestDone = false;
    this.exchangeResponseDone = false;
    return result;
  }
}
```

这里调用了 ExchangeFinder 的 find 方法用以查找一个可用的连接，ExchangeFinder 在 RetryAndFollowUpInterceptor 调用 Transmitter 的 prepareToConnect 方法时创建的。随后创建了 Exchange 并返回。来看看 find 方法：

```java
public ExchangeCodec find(
    OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
  int connectTimeout = chain.connectTimeoutMillis();
  int readTimeout = chain.readTimeoutMillis();
  int writeTimeout = chain.writeTimeoutMillis();
  int pingIntervalMillis = client.pingIntervalMillis();
  boolean connectionRetryEnabled = client.retryOnConnectionFailure();

  try {
    // 查找可用连接，如果连接不可用，会一直重试
    RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
        writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
    return resultConnection.newCodec(client, chain);
  } catch (RouteException e) {
    trackFailure();
    throw e;
  } catch (IOException e) {
    trackFailure();
    throw new RouteException(e);
  }
}

...
  
/**
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      // 查找连接
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0 && !candidate.isMultiplexed()) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        candidate.noNewExchanges();
        continue;
      }

      return candidate;
    }
  }

...
  
  /**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    RealConnection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (transmitter.isCanceled()) throw new IOException("Canceled");
      hasStreamFailure = false; // This is a fresh attempt.

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new exchanges.
      releasedConnection = transmitter.connection;
      toClose = transmitter.connection != null && transmitter.connection.noNewExchanges
          ? transmitter.releaseConnectionNoEvents()
          : null;

      if (transmitter.connection != null) {
        // We had an already-allocated connection and it's good.
        result = transmitter.connection;
        releasedConnection = null;
      }

      if (result == null) {
        // Attempt to get a connection from the pool.
        if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, null, false)) {
          foundPooledConnection = true;
          result = transmitter.connection;
        } else if (nextRouteToTry != null) {
          selectedRoute = nextRouteToTry;
          nextRouteToTry = null;
        } else if (retryCurrentRoute()) {
          selectedRoute = transmitter.connection.route();
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    List<Route> routes = null;
    synchronized (connectionPool) {
      if (transmitter.isCanceled()) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        routes = routeSelection.getAll();
        if (connectionPool.transmitterAcquirePooledConnection(
            address, transmitter, routes, false)) {
          foundPooledConnection = true;
          result = transmitter.connection;
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        result = new RealConnection(connectionPool, selectedRoute);
        connectingConnection = result;
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    connectionPool.routeDatabase.connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      connectingConnection = null;
      // Last attempt at connection coalescing, which only occurs if we attempted multiple
      // concurrent connections to the same host.
      if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, routes, true)) {
        // We lost the race! Close the connection we created and return the pooled connection.
        result.noNewExchanges = true;
        socket = result.socket();
        result = transmitter.connection;

        // It's possible for us to obtain a coalesced connection that is immediately unhealthy. In
        // that case we will retry the route we just successfully connected with.
        nextRouteToTry = selectedRoute;
      } else {
        connectionPool.put(result);
        transmitter.acquireConnectionNoEvents(result);
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```

查找可用连接的优先级：**当前连接 > 连接池里的连接 > 创建新连接**。findConnection 方法就是执行此流程，直到找到一个可用连接返回。

> 当前连接可用则直接使用它，否则从连接池中获取连接，如果连接池中没有则创建一个新的连接，并执行 TCP + TLS 握手，并将新连接放入连接池中。

找可用连接后就会执行 CallServerInterceptor。

## CallServerInterceptor

```java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Exchange exchange = realChain.exchange();
  Request request = realChain.request();

  long sentRequestMillis = System.currentTimeMillis();

  // 执行写入请求头
  exchange.writeRequestHeaders(request);

  boolean responseHeadersStarted = false;
  Response.Builder responseBuilder = null;
  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
    // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
    // Continue" response before transmitting the request body. If we don't get that, return
    // what we did get (such as a 4xx response) without ever transmitting the request body.
    if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
      exchange.flushRequest();
      responseHeadersStarted = true;
      exchange.responseHeadersStart();
      responseBuilder = exchange.readResponseHeaders(true);
    }

    // 执行写入请求体
    if (responseBuilder == null) {
      if (request.body().isDuplex()) {
        // Prepare a duplex body so that the application can send a request body later.
        exchange.flushRequest();
        BufferedSink bufferedRequestBody = Okio.buffer(
            exchange.createRequestBody(request, true));
        request.body().writeTo(bufferedRequestBody);
      } else {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        BufferedSink bufferedRequestBody = Okio.buffer(
            exchange.createRequestBody(request, false));
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
      }
    } else {
      exchange.noRequestBody();
      if (!exchange.connection().isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        exchange.noNewExchangesOnConnection();
      }
    }
  } else {
    exchange.noRequestBody();
  }

  if (request.body() == null || !request.body().isDuplex()) {
    exchange.finishRequest();
  }

  if (!responseHeadersStarted) {
    exchange.responseHeadersStart();
  }

  if (responseBuilder == null) {
    // 读取响应头
    responseBuilder = exchange.readResponseHeaders(false);
  }

  Response response = responseBuilder
      .request(request)
      .handshake(exchange.connection().handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();

  int code = response.code();
  if (code == 100) {
    // server sent a 100-continue even though we did not request one.
    // try again to read the actual response
    response = exchange.readResponseHeaders(false)
        .request(request)
        .handshake(exchange.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    code = response.code();
  }

  exchange.responseHeadersEnd(response);

  if (forWebSocket && code == 101) {
    // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
    response = response.newBuilder()
        .body(Util.EMPTY_RESPONSE)
        .build();
  } else {
    // 读取响应体
    response = response.newBuilder()
        .body(exchange.openResponseBody(response))
        .build();
  }

  if ("close".equalsIgnoreCase(response.request().header("Connection"))
      || "close".equalsIgnoreCase(response.header("Connection"))) {
    exchange.noNewExchangesOnConnection();
  }

  if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
    throw new ProtocolException(
        "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
  }

  // 返回响应
  return response;
}
```

CallServerInterceptor 的主要作用：

1、写入请求头

2、写入请求体

3、读取响应头

4、读取响应体

5、返回响应

