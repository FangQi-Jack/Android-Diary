### LifecycleOwner

```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

`LifecycleOwner` 是一个接口，定义了 `getLifeCycle` 方法。`ComponentActivity` 和 `Fragment` 都实现了该接口。

#### ComponentActivity 中的 Lifecycle

```java
private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

@NonNull
@Override
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
```

可以看到，在 `ComponentActivity` 的 `getLifecycle` 中直接返回了 `mLifecycleRegistry` 对象，它是 `LifecycleRegistry` 类型的对象。

```java
public LifecycleRegistry(@NonNull LifecycleOwner provider) {
    this(provider, true);
}

private LifecycleRegistry(@NonNull LifecycleOwner provider, boolean enforceMainThread) {
    mLifecycleOwner = new WeakReference<>(provider);
    mState = INITIALIZED;
    mEnforceMainThread = enforceMainThread;
}
```

在 `LifecycleRegistry` 的构造函数中，将传入的 `LifecycleOwner` 放入弱引用中保存。那么 `ComponentActivity` 的生命周期又是如何与 `LifecycleRegistry` 关联起来的呢？

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ...
    ReportFragment.injectIfNeededIn(this);
    ...
}
```

在 `ComponentActivity` 的 `onCreate` 方法中，有一行重要代码 `ReportFragment.injectIfNeededIn(this);`，正是在  `ReportFragment` 中实现了 `ComponentActivity` 的生命周期分发到 `LifecycleRegistry` 中。接下来看看 `injectIfNeededIn` 方法。

```java
public class ReportFragment extends android.app.Fragment {
    private static final String REPORT_FRAGMENT_TAG = "androidx.lifecycle"
            + ".LifecycleDispatcher.report_fragment_tag";
  
  	public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            // On API 29+, we can register for the correct Lifecycle callbacks directly
            LifecycleCallbacks.registerIn(activity);
        }
        // Prior to API 29 and to maintain compatibility with older versions of
        // ProcessLifecycleOwner (which may not be updated when lifecycle-runtime is updated and
        // need to support activities that don't extend from FragmentActivity from support lib),
        // use a framework fragment to get the correct timing of Lifecycle events
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }
}
```

对于 API 29 及以上，会执行 `LifecycleCallbacks` 的 `registerIn` 方法，然后将 `ReportFragment` 添加到 `Activity` 中。`LifecycleCallbacks` 实现了 `Application.ActivityLifecycleCallbacks` 接口，在 `Activity` 生命周期回调方法中会执行 `ReportFragment` 的 `dispatch` 方法。

```java
@RequiresApi(29)
static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {

    static void registerIn(Activity activity) {
        activity.registerActivityLifecycleCallbacks(new LifecycleCallbacks());
    }

    @Override
    public void onActivityCreated(@NonNull Activity activity,
            @Nullable Bundle bundle) {
    }

    @Override
    public void onActivityPostCreated(@NonNull Activity activity,
            @Nullable Bundle savedInstanceState) {
        dispatch(activity, Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onActivityStarted(@NonNull Activity activity) {
    }

    @Override
    public void onActivityPostStarted(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_START);
    }

    @Override
    public void onActivityResumed(@NonNull Activity activity) {
    }

    @Override
    public void onActivityPostResumed(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onActivityPrePaused(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onActivityPaused(@NonNull Activity activity) {
    }

    @Override
    public void onActivityPreStopped(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onActivityStopped(@NonNull Activity activity) {
    }

    @Override
    public void onActivitySaveInstanceState(@NonNull Activity activity,
            @NonNull Bundle bundle) {
    }

    @Override
    public void onActivityPreDestroyed(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_DESTROY);
    }

    @Override
    public void onActivityDestroyed(@NonNull Activity activity) {
    }
}
```

而在 `ReportFragment` 的各个生命周期方法中也会调用 `dispatch` 方法，最终执行的方法与上述一致。

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatchCreate(mProcessListener);
    dispatch(Lifecycle.Event.ON_CREATE);
}

@Override
public void onStart() {
    super.onStart();
    dispatchStart(mProcessListener);
    dispatch(Lifecycle.Event.ON_START);
}

@Override
public void onResume() {
    super.onResume();
    dispatchResume(mProcessListener);
    dispatch(Lifecycle.Event.ON_RESUME);
}

@Override
public void onPause() {
    super.onPause();
    dispatch(Lifecycle.Event.ON_PAUSE);
}

@Override
public void onStop() {
    super.onStop();
    dispatch(Lifecycle.Event.ON_STOP);
}

@Override
public void onDestroy() {
    super.onDestroy();
    dispatch(Lifecycle.Event.ON_DESTROY);
    // just want to be sure that we won't leak reference to an activity
    mProcessListener = null;
}

private void dispatch(@NonNull Lifecycle.Event event) {
    if (Build.VERSION.SDK_INT < 29) {
        // Only dispatch events from ReportFragment on API levels prior
        // to API 29. On API 29+, this is handled by the ActivityLifecycleCallbacks
        // added in ReportFragment.injectIfNeededIn
        dispatch(getActivity(), event);
    }
}
```

那么我们看看 `dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event)` 方法的实现：

```java
@SuppressWarnings("deprecation")
static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

其中 `LifecycleRegistryOwner` 是一个继承自 `LifecycleOwner` 的接口，所以，最终都会执行 `LifecycleRegistry` 的 `handleLifecyleEvent` 方法。

```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    enforceMainThreadIfNeeded("handleLifecycleEvent");
    moveToState(event.getTargetState());
}

private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}

private boolean isSynced() {
  if (mObserverMap.size() == 0) {
    	return true;
  }
  State eldestObserverState = mObserverMap.eldest().getValue().mState;
  State newestObserverState = mObserverMap.newest().getValue().mState;
  return eldestObserverState == newestObserverState && mState == newestObserverState;
}

private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
      	throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                                      + "garbage collected. It is too late to change lifecycle state.");
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
          	backwardPass(lifecycleOwner);
        }
        Map.Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
            && mState.compareTo(newest.getValue().mState) > 0) {
          	forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}

private void forwardPass(LifecycleOwner lifecycleOwner) {
  Iterator<Map.Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
    	mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Map.Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            final Event event = Event.upFrom(observer.mState);
            if (event == null) {
                throw new IllegalStateException("no event up from " + observer.mState);
            }
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}

private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Map.Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
      		mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Map.Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = Event.downFrom(observer.mState);
            if (event == null) {
              	throw new IllegalStateException("no event down from " + observer.mState);
            }
            pushParentState(event.getTargetState());
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}

static class ObserverWithState {
    State mState;
    LifecycleEventObserver mLifecycleObserver;
		...
    void dispatchEvent(LifecycleOwner owner, Event event) {
        ...
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

在 `handleLifecycleEvent` 中会执行 `moveToState` 方法，并传入当前生命周期状态，在 `moveToState` 方法中会将当前生命周期状态赋值为新的状态，接着会执行 `sync` 方法，在该方法中会执行遍历比较添加的 `LifecycleObserver` 的状态和当前状态，最终会执行 `ObserverWithState` 的 `dispatchEvent` 方法，此方法中会回调 `LifecycleObserver` 的 `onStateChanged` 方法。



#### Fragment 中的 Lifecycle

```java
LifecycleRegistry mLifecycleRegistry;

@Override
@NonNull
public Lifecycle getLifecycle() {
  	return mLifecycleRegistry;
}

public Fragment() {
  	initLifecycle();
}

private void initLifecycle() {
    mLifecycleRegistry = new LifecycleRegistry(this);
    mSavedStateRegistryController = SavedStateRegistryController.create(this);
    if (Build.VERSION.SDK_INT >= 19) {
        mLifecycleRegistry.addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                                       @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_STOP) {
                    if (mView != null) {
                      	mView.cancelPendingInputEvents();
                    }
                }
            }
        });
    }
}

void performCreate(Bundle savedInstanceState) {
    ...
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
}

void performStart() {
  	...
    mChildFragmentManager.dispatchStart();
}

void performResume() {
    ...
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
    if (mView != null) {
      mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
    }
    mChildFragmentManager.dispatchResume();
    ...
}

void performPause() {
  	...
    if (mView != null) {
      	mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE);
    }
  	mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE);
  	...
}

void performStop() {
  	...
    if (mView != null) {
      	mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_STOP);
    }
  	mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP);
  	...
}

void performDestroyView() {
  	...
    if (mView != null) {
      mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
    }
  	...
}

void performDestroy() {
    ...
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
    ...
}
```

在 `Fragment` 的构造方法中，会调用 `initLifecycle` 方法初始化 `mLifecycleRegistry` ，在 `getLifecycle` 中返回。在 `Fragment` 的各个生命周期方法中同样会调用 `LifecycleRegistry` 的 `handleLifecycleEvent` 方法进行生命周期事件分发。



### LifecycleObserver

```java
public interface LifecycleEventObserver extends LifecycleObserver {
    /**
     * Called when a state transition event happens.
     *
     * @param source The source of the event
     * @param event The event
     */
    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}
```

`LifecycleObserver` 是一个空接口。`LifecycleEventObserver` 继承了它，并定义了 `onStateChanged` 方法，该方法在生命周期变化时会被回调。`LifecycleRegistry` 的 `addObserver` 方法：

```java
private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();

@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    enforceMainThreadIfNeeded("addObserver");
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    ...
}
```

在该方法中，首先创建了一个 `ObserverWithState` 类型的 `statefulObserver` 。然后执行了 `mObserverMap` 的 `putIfAbsent` 方法，该方法会取出当前 `key` 对应的 `Entry`，如果 `Entry` 不为空则返回它对应的 `value`，否则将新的键值对存入并返回 null。

