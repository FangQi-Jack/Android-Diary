# OkHttp 的连接池

在 ConnectInterceptor 查找可用连接时，会尝试从连接池中查找。连接复用省去了 TCP + TSL 握手，提高了效率。

## RealConnectionPool

RealConnectionPool 在 ConnectionPool 的构造函数中被构造。

RealConnectionPool 中维护了一个 RealConnection 的双向队列、失败路由黑名单。

```java
public final class ConnectionPool {
  final RealConnectionPool delegate;

  /**
   * Create a new connection pool with tuning parameters appropriate for a single-user application.
   * The tuning parameters in this pool are subject to change in future OkHttp releases. Currently
   * this pool holds up to 5 idle connections which will be evicted after 5 minutes of inactivity.
   */
  public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    // 构造 RealConnectionPool
    this.delegate = new RealConnectionPool(maxIdleConnections, keepAliveDuration, timeUnit);
  }
  ...
}
```

```java
public final class RealConnectionPool {
  /**
   * Background threads are used to cleanup expired connections. There will be at most a single
   * thread running per connection pool. The thread pool executor permits the pool itself to be
   * garbage collected.
   */
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<>(), Util.threadFactory("OkHttp ConnectionPool", true));

  /** The maximum number of idle connections for each address. */
  private final int maxIdleConnections;
  // socket 的 keepAlive 时长
  private final long keepAliveDurationNs;
  // 清理
  private final Runnable cleanupRunnable = () -> {
    while (true) {
      // 执行清理任务
      long waitNanos = cleanup(System.nanoTime());
      if (waitNanos == -1) return;
      if (waitNanos > 0) {
        long waitMillis = waitNanos / 1000000L;
        waitNanos -= (waitMillis * 1000000L);
        synchronized (RealConnectionPool.this) {
          try {
            RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
          } catch (InterruptedException ignored) {
          }
        }
      }
    }
  };

  // 维护 RealConnection 的 socket 连接的双向队列
  private final Deque<RealConnection> connections = new ArrayDeque<>();
  // 连接失败的路由黑名单
  final RouteDatabase routeDatabase = new RouteDatabase();
  // 是否正在执行清理工作
  boolean cleanupRunning;
  
  ...
    
  /**
   * Attempts to acquire a recycled connection to {@code address} for {@code transmitter}. Returns
   * true if a connection was acquired.
   *
   * <p>If {@code routes} is non-null these are the resolved routes (ie. IP addresses) for the
   * connection. This is used to coalesce related domains to the same HTTP/2 connection, such as
   * {@code square.com} and {@code square.ca}.
   */
  boolean transmitterAcquirePooledConnection(Address address, Transmitter transmitter,
      @Nullable List<Route> routes, boolean requireMultiplexed) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (requireMultiplexed && !connection.isMultiplexed()) continue;
      if (!connection.isEligible(address, routes)) continue;
      transmitter.acquireConnectionNoEvents(connection);
      return true;
    }
    return false;
  }
  
  // 放入连接
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      // 执行清理任务
      executor.execute(cleanupRunnable);
    }
    // 将连接放入队列中
    connections.add(connection);
  }
  
  ...
    
    /**
   * Performs maintenance on this pool, evicting the connection that has been idle the longest if
   * either it has exceeded the keep alive limit or the idle connections limit.
   *
   * <p>Returns the duration in nanos to sleep until the next scheduled call to this method. Returns
   * -1 if no further cleanups are required.
   */
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }
  
  /**
   * Prunes any leaked transmitters and then returns the number of remaining live transmitters on
   * {@code connection}. Transmitters are leaked if the connection is tracking them but the
   * application code has abandoned them. Leak detection is imprecise and relies on garbage
   * collection.
   */
  private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    // 遍历弱引用集合
    List<Reference<Transmitter>> references = connection.transmitters;
    for (int i = 0; i < references.size(); ) {
      Reference<Transmitter> reference = references.get(i);

      if (reference.get() != null) {
        i++;
        continue;
      }

      // We've discovered a leaked transmitter. This is an application bug.
      TransmitterReference transmitterRef = (TransmitterReference) reference;
      String message = "A connection to " + connection.route().address().url()
          + " was leaked. Did you forget to close a response body?";
      Platform.get().logCloseableLeak(message, transmitterRef.callStackTrace);

      // 移除引用
      references.remove(i);
      connection.noNewExchanges = true;

      // If this was the last allocation, the connection is eligible for immediate eviction.
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }

    return references.size();
  }
}
```

默认情况下，maxIdleConnections = 5，keepAliveDurationNs = 5 分钟。

### 放入队列

put 方法中主要做两件事：1、如果没有执行清理则开始执行清理任务。2、将连接放入队列中。

### 清理

1、遍历连接队列

2、根据连接的引用数来统计出活跃连接和空闲连接数

3、找到空闲时间最久的连接

4、如果空闲时间大于最大空闲时间限制或这个空闲个数超过最大空闲个数限制，将空闲时间最久的连接清理掉

5、如果有空闲连接，返回空闲最久的连接的剩余时间

6、如果活跃连接数大于 0，返回最长存活时间



# Tansmitter

Transmitter 类中处理连接引用数的增减。在 ConnectInterceptor 查找可用连接时也调用了一下两个方法用以清理引用和增加引用。

```java
public final class Transmitter {
  ...
  
  void acquireConnectionNoEvents(RealConnection connection) {
    assert (Thread.holdsLock(connectionPool));

    if (this.connection != null) throw new IllegalStateException();
    this.connection = connection;
    // 添加一个引用
    connection.transmitters.add(new TransmitterReference(this, callStackTrace));
  }
  
    /**
   * Remove the transmitter from the connection's list of allocations. Returns a socket that the
   * caller should close.
   */
  @Nullable Socket releaseConnectionNoEvents() {
    assert (Thread.holdsLock(connectionPool));

    int index = -1;
    // 遍历找到引用下标
    for (int i = 0, size = this.connection.transmitters.size(); i < size; i++) {
      Reference<Transmitter> reference = this.connection.transmitters.get(i);
      if (reference.get() == this) {
        index = i;
        break;
      }
    }

    if (index == -1) throw new IllegalStateException();

    RealConnection released = this.connection;
    // 移除引用
    released.transmitters.remove(index);
    this.connection = null;

    if (released.transmitters.isEmpty()) {
      released.idleAtNanos = System.nanoTime();
      if (connectionPool.connectionBecameIdle(released)) {
        return released.socket();
      }
    }

    return null;
  }
  
  ...
    
  static final class TransmitterReference extends WeakReference<Transmitter> {
    /**
     * Captures the stack trace at the time the Call is executed or enqueued. This is helpful for
     * identifying the origin of connection leaks.
     */
    final Object callStackTrace;

    TransmitterReference(Transmitter referent, Object callStackTrace) {
      super(referent);
      this.callStackTrace = callStackTrace;
    }
  }
}
```