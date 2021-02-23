## 简单使用

```kotlin
Observable.create(
    ObservableOnSubscribe<Int> { emitter ->
        emitter.onNext(1)
        emitter.onComplete()
    }
).subscribe(object : Observer<Int> {
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

## subscribe

```java
public abstract class Observable<T> implements ObservableSource<T> {
  	...
  
    @SchedulerSupport(SchedulerSupport.NONE)
    @Override
    public final void subscribe(Observer<? super T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);

            ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");

            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // can't call onError because no way to know if a Disposable has been set or not
            // can't call onSubscribe because the call might have set a Subscription already
            RxJavaPlugins.onError(e);

            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }

    /**
     * Operator implementations (both source and intermediate) should implement this method that
     * performs the necessary business logic and handles the incoming {@link Observer}s.
     * <p>There is no need to call any of the plugin hooks on the current {@code Observable} instance or
     * the {@code Observer}; all hooks and basic safeguards have been
     * applied by {@link #subscribe(Observer)} before this method gets called.
     * @param observer the incoming Observer, never null
     */
    protected abstract void subscribeActual(Observer<? super T> observer)
      
    ...
}
```

可以看到在 `subscribe` 方法中调用了 `subscribeActual(observer)` 方法，从方法名可以看出，此方法事件完成了订阅。`subscribeActual(observer)` 是一个抽象方法，其实现类如 `ObservableCreate`。

#### ObservableCreate.subscribeActual

```java
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;
  	
  	...
      
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
  
  	...
      
    static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
      	private static final long serialVersionUID = -3434801548987643227L;

        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }

        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }

        @Override
        public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }
      
      	@Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }
      
      	@Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
}
```

在 `ObservableCreate` 的 `subscribeActual` 方法中，首先创建了 `CreateEmitter` 类型的发射器，并将 `observer` 传递给了发射器，然后回调了 `observer` 的 `onSubscribe` 方法，并把发射器（此处作为 `Disposable`）传给了 `observer`，最后调用了 `ObservableOnSubscribe` 的 `subscribe` 方法，继续给上游的 `ObservableOnSubscribe` 添加订阅，并且将 `observer` 作为参数传递进去。

## Emitter 发送数据

在 `CreateEmitter` 的 `onNext` 方法中判断了是否已经 `disposed` ，如果为 `false` 则回调 `observer` 的 `onNext` 方法，数据发送到 `observer`，在 `CreateEmitter`的 `onComplete` 方法中，调用了 `observer` 的 `onComplete` 并执行了 `dispose` 方法，在 `dispose` 方法中会将 `emitter` 设置为 `DISPOSED`，此后该发射器将不再能发送数据。