```kotlin
Observable.create(
  ObservableOnSubscribe<Int> { emitter ->
                              emitter.onNext(1)
                              emitter.onComplete()
                             }
).subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(object : Observer<Int> {
  override fun onSubscribe(d: Disposable) {

  }

  override fun onNext(t: Int) {

  }

  override fun onError(e: Throwable) {

  }

  override fun onComplete() {

  }
})
```

## Observable.subscribeOn

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```

在 `Observable` 的 `subscribeOn` 方法中，返回了一个 `ObservableSubscribeOn` 对象。

```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

        observer.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }

    static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

        private static final long serialVersionUID = 8094547886072529208L;
        final Observer<? super T> downstream;

        final AtomicReference<Disposable> upstream;

        SubscribeOnObserver(Observer<? super T> downstream) {
            this.downstream = downstream;
            this.upstream = new AtomicReference<Disposable>();
        }

        @Override
        public void onSubscribe(Disposable d) {
            DisposableHelper.setOnce(this.upstream, d);
        }

        @Override
        public void onNext(T t) {
            downstream.onNext(t);
        }

        @Override
        public void onError(Throwable t) {
            downstream.onError(t);
        }

        @Override
        public void onComplete() {
            downstream.onComplete();
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(upstream);
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }

        void setDisposable(Disposable d) {
            DisposableHelper.setOnce(this, d);
        }
    }

    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
}
```

`ObservableSubscribeOn` 是一个 `Observable` 类型的对象，在它的 `subscribeActual` 方法中，首先会创建 `SubscribeOnObserver` 对象，并将 `observer` 作为参数传递进去，`SubscribeOnObserver` 实现了 `Observer` 和 `Disposable`，随后执行了 `observer` 的 `onSubscribe` 方法，并将 `SubscribeOnObserver` 作为 `Disposable` 类型的对象传过去，最后给 `SubscribeOnObserver` 设置了 `Disposable`。在示例代码中此处的 `scheduler` 为 `IoScheduler`，而 `SubscribeTask` 是一个 `Runnable` 类型的对象，因此最终会通过 `IoScheduler` 的线程池执行它的 `run` 方法，在此方法内将完成上游 `Observable` 的添加订阅的工作。



## Observable.observeOn

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, false, bufferSize());
}

@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```

`observeOn` 方法返回了一个 `ObservableObserveOn` 对象。

```java
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
		final Scheduler scheduler;
  
  	@Override
    protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            Scheduler.Worker w = scheduler.createWorker();

            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }
}
```

在 `ObservableObserveOn` 的 `subscribeActual` 方法中，先通过 `scheduler` （此处为 `HandlerScheduler`）获取 `Worker`，它是 `HandlerWorker` 类型的对象，随后执行了 `ObservableSource` 的 `subscribe` 方法，并在传参中创建了 `ObserveOnObserver`。

```java
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
implements Observer<T>, Runnable {
  	final Observer<? super T> downstream;
    final Scheduler.Worker worker;
    final boolean delayError;
    final int bufferSize;
  	
  	ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
        this.downstream = actual;
        this.worker = worker;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }
  
  	@Override
    public void onSubscribe(Disposable d) {
      if (DisposableHelper.validate(this.upstream, d)) {
          this.upstream = d;
          ...

          queue = new SpscLinkedArrayQueue<T>(bufferSize);

          downstream.onSubscribe(this);
      }
    }

    @Override
    public void onNext(T t) {
      if (done) {
        return;
      }

      if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
      }
      schedule();
    }
  
  	@Override
    public void run() {
        if (outputFused) {
          	drainFused();
        } else {
          	drainNormal();
        }
    }
  
  	void drainFused() {
        int missed = 1;

        for (;;) {
          if (disposed) {
            	return;
          }

          boolean d = done;
          Throwable ex = error;

          if (!delayError && d && ex != null) {
              disposed = true;
              downstream.onError(error);
              worker.dispose();
              return;
          }

          downstream.onNext(null);

          if (d) {
              disposed = true;
              ex = error;
              if (ex != null) {
                	downstream.onError(ex);
              } else {
                	downstream.onComplete();
              }
              worker.dispose();
              return;
          }

          missed = addAndGet(-missed);
          if (missed == 0) {
            	break;
          }
        }
      }
}
```

在 `ObserveOnObserver` 的 `onSubscribe` 方法中会执行下游的 `onSubscribe` 方法回调，而在 `onNext` 方法中会调用 `HandlerWorker` 的 `schedule` 方法，在 `HandlerWorker` 的 `schedule` 方法中通过 `UI` 线程的 `Handler` 发送了一条消息，而 `ObserveOnObserver` 实现了 `Runnable` ，因此将在 `UI` 线程执行 `run` 方法。从源码可以看到，在 `run` 方法中，会执行下游的 `onNext`、`onError` 等方法。