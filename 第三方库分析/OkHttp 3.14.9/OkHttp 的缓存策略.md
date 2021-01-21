# **OkHttp 的缓存策略**

### Http 缓存机制

##### Http 缓存控制

> Http1.0：**Expires**
>
> Http1.1：**Cache-Control**
>
> Expires 表示缓存过期时间，Cache-Control 表示缓存最大存活时间。如果二者同时存在，以 Cache-Control 为主。

> Http1.0：**Last-Modified** && **If-Modified-Since**
>
> Http1.1：**Etag** && **If-None-Match**
>
> Last-Modified 由服务器响应头携带，表示资源文件在服务器最后被修改的时间。
>
> > 请求流程：首次请求服务器，服务在响应头中携带 Last-Modified 信息，随后客户端每次请求服务器，请求头中都会带上 If-Modified-Since，它的取值就是上一次服务器响应头中的 Last-Modified，服务器收到请求后，将 If-Modified-Since 的值服务器上的最后修改时间对比，判断资源是否变化。如果发生变化则返回200，新的资源内容和新的 Last-Modified，否则返回 304和新的 Last-Modified。
>
> Etag 是服务器生成的资源文件的唯一标识。
>
> > 请求流程：首次请求服务器，服务器在响应头中携带 Etag，随后客户端每次请求服务器都在请求头中带上 If-None-Match，它等于服务器上一次响应头中的 Etag，服务器收到请求对资源标识符作对比判断资源是否发生改变，如果资源变化了则会返回新的内容和新的 Etag，否则返回 304 并且响应头中不会携带 Etag。



### OkHttp 的缓存策略

OkHttp 的缓存策略主要是由 CacheStrategy 中的 `networkRequest` 和 `cacheResponse` 来决定的。

| networkRequest | cacheResponse | 结果                                                         |
| -------------- | ------------- | ------------------------------------------------------------ |
| not-null       | null          | 请求网络                                                     |
| null           | not-null      | 使用缓存                                                     |
| not-null       | not-null      | 响应头包含Etag或Last-Modified，需要网络请求进行验证是否可以使用缓存 |
| null           | null          | 禁止使用网络和缓存                                           |

在 CacheInterceptor 的 intercept 方法中会通过 CacheStrategy.Factory 的 get 方法获取缓存策略：

```java
/**
 * Returns a strategy to satisfy {@code request} using the a cached response {@code response}.
 */
public CacheStrategy get() {
  CacheStrategy candidate = getCandidate();

  if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
    // We're forbidden from using the network and the cache is insufficient.
    return new CacheStrategy(null, null);
  }

  return candidate;
}
```

其中通过调用 getCandidate 方法获取了 CacheStrategy：

```java
/** Returns a strategy to use assuming the request can use the network. */
private CacheStrategy getCandidate() {
  // No cached response.
  if (cacheResponse == null) {
    // 使用网络
    return new CacheStrategy(request, null);
  }

  // Drop the cached response if it's missing a required handshake.
  if (request.isHttps() && cacheResponse.handshake() == null) {
    // 使用网络
    return new CacheStrategy(request, null);
  }

  // If this response shouldn't have been stored, it should never be used
  // as a response source. This check should be redundant as long as the
  // persistence store is well-behaved and the rules are constant.
  if (!isCacheable(cacheResponse, request)) {
    // 使用网络
    return new CacheStrategy(request, null);
  }

  // 获取请求头中的 CacheControl
  CacheControl requestCaching = request.cacheControl();
  // 请求头设置不缓存或者请求头包含 If-Modified-Since 或 If-None-Match，意味着缓存失效，需要通过服务器验证
  if (requestCaching.noCache() || hasConditions(request)) {
   	// 使用网络
    return new CacheStrategy(request, null);
  }

  // 获取缓存响应头
  CacheControl responseCaching = cacheResponse.cacheControl();

  // 计算缓存响应的年龄
  long ageMillis = cacheResponseAge();
  // 计算响应的刷新时间
  long freshMillis = computeFreshnessLifetime();

  if (requestCaching.maxAgeSeconds() != -1) {
    // 取刷新时间和年龄中较小的
    freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
  }

  long minFreshMillis = 0;
  
  if (requestCaching.minFreshSeconds() != -1) {
    // 更新最小刷新时间的限制
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
  }

  long maxStaleMillis = 0;
  if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
    // 更新最大验证时间
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
  }

  // 可以缓存并且 年龄+最小刷新时间 < 刷新时间+最大验证时间。意味着缓存虽然过期，但可用
  if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    Response.Builder builder = cacheResponse.newBuilder();
    if (ageMillis + minFreshMillis >= freshMillis) {
      builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
    }
    long oneDayMillis = 24 * 60 * 60 * 1000L;
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
      builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
    }
    // 使用缓存
    return new CacheStrategy(null, builder.build());
  }

  // Find a condition to add to the request. If the condition is satisfied, the response body
  // will not be transmitted.
  String conditionName;
  String conditionValue;
  if (etag != null) {
    conditionName = "If-None-Match";
    conditionValue = etag;
  } else if (lastModified != null) {
    conditionName = "If-Modified-Since";
    conditionValue = lastModifiedString;
  } else if (servedDate != null) {
    conditionName = "If-Modified-Since";
    conditionValue = servedDateString;
  } else {
    // 使用网络
    return new CacheStrategy(request, null); // No condition! Make a regular request.
  }

  Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
  Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

  Request conditionalRequest = request.newBuilder()
      .headers(conditionalRequestHeaders.build())
      .build();
  // 使用网络 + 缓存
  return new CacheStrategy(conditionalRequest, cacheResponse);
}
```

**使用网络：**

* 没有缓存
* https 丢失了握手
* 不能缓存响应
* 请求头设置不缓存或者请求头包含 If-Modified-Since 或 If-None-Match
* 响应头中没有 Etag 和 Last-Modified

**使用缓存**

* 年龄+最小刷新时间 < 刷新时间+最大验证时间

**使用网络和缓存**

* 响应头包含 Etag 或 Last-Modified

**禁止使用网络和缓存**

* 网络请求策略不为 null，但是请求头的 cache-control 设置了只用缓存