### 使用

```kotlin
private val mViewModel by lazy {
    ViewModelProvider(this).get(MainViewModel::class.java)
}
```

### ViewModelProvider

`ViewModelProvider` ，顾名思义，就是提供 `ViewModel` 的类。

```java
public class ViewModelProvider {

    private static final String DEFAULT_KEY = "androidx.lifecycle.ViewModelProvider.DefaultKey";
  	
  	/**
     * Implementations of {@code Factory} interface are responsible to instantiate ViewModels.
     */
    public interface Factory {
        /**
         * Creates a new instance of the given {@code Class}.
         * <p>
         *
         * @param modelClass a {@code Class} whose instance is requested
         * @param <T>        The type parameter for the ViewModel.
         * @return a newly created ViewModel
         */
        @NonNull
        <T extends ViewModel> T create(@NonNull Class<T> modelClass);
    }
  
  	...
      
    private final Factory mFactory;
    private final ViewModelStore mViewModelStore;
  
  	...

    public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }

    public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }

    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
        } else {
            viewModel = mFactory.create(modelClass);
        }
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }
  
  	...
}
```

在 `ViewModelProvider` 的构造函数中保存了 `ViewModelStore` 和 `ViewModelProvider.Factory` 。在 `get` 方法中，通过传入的 `Class` 构造出 key，然后通过 key 和 `Class` 获取 `ViewModel` 对象，首先通过 key 在 `mViewModelStore` 中查找，如果找到了且与传入的 `Class` 相符则直接返回，否则调用 `ViewModelProvider.Factory` 的 `create` 方法创建新的 `ViewModel` 并将其存入 `mViewModelStore` 中，然后返回新创建的 `ViewModel` 对象。



### Factory

```java
/**
 * Simple factory, which calls empty constructor on the give class.
 */
public static class NewInstanceFactory implements Factory {

    private static NewInstanceFactory sInstance;

    /**
     * Retrieve a singleton instance of NewInstanceFactory.
     *
     * @return A valid {@link NewInstanceFactory}
     */
    @NonNull
    static NewInstanceFactory getInstance() {
        if (sInstance == null) {
            sInstance = new NewInstanceFactory();
        }
        return sInstance;
    }

    @SuppressWarnings("ClassNewInstance")
    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        //noinspection TryWithIdenticalCatches
        try {
            return modelClass.newInstance();
        } catch (InstantiationException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        }
    }
}

/**
 * {@link Factory} which may create {@link AndroidViewModel} and
 * {@link ViewModel}, which have an empty constructor.
 */
public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {

    private static AndroidViewModelFactory sInstance;

    /**
     * Retrieve a singleton instance of AndroidViewModelFactory.
     *
     * @param application an application to pass in {@link AndroidViewModel}
     * @return A valid {@link AndroidViewModelFactory}
     */
    @NonNull
    public static AndroidViewModelFactory getInstance(@NonNull Application application) {
        if (sInstance == null) {
            sInstance = new AndroidViewModelFactory(application);
        }
        return sInstance;
    }

    private Application mApplication;

    /**
     * Creates a {@code AndroidViewModelFactory}
     *
     * @param application an application to pass in {@link AndroidViewModel}
     */
    public AndroidViewModelFactory(@NonNull Application application) {
        mApplication = application;
    }

    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
            //noinspection TryWithIdenticalCatches
            try {
                return modelClass.getConstructor(Application.class).newInstance(mApplication);
            } catch (NoSuchMethodException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (InstantiationException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
        return super.create(modelClass);
    }
}
```

系统提供了两种 `ViewModelProvider.Factory` 的实现：`NewInstanceFactory` 和 `AndroidViewModelFactory`。其中 `NewInstanceFactory` 比较简单，其 `create` 方法中就是直接创建对象。而 `AndroidViewModelFactory` 继承自 `NewInstanceFactory`，在它的 `create` 方法中创建对象时创了 `mAppliation` 参数。



### ViewModelStore

顾名思义，就是用来存储 `ViewModel` 的。

```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

在 `ViewModelStore` 中用 `Map` 存储 `ViewModel`，并对外提供了 `clear` 方法。那么，它的 `clear` 方法是在什么时候被调用的呢？

##### 1、ComponentActivity 中被调用

```java
public ComponentActivity() {
  	getLifecycle().addObserver(new LifecycleEventObserver() {
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                                   @NonNull Lifecycle.Event event) {
          if (event == Lifecycle.Event.ON_DESTROY) {
              if (!isChangingConfigurations()) {
                	getViewModelStore().clear();
              }
          }
        }
    });
}
```

可见，当 `Activity` 不是因为配置变化而被销毁时，会执行 `ViewModelStore` 的 `clear` 方法。由此也可知，当 `Activity` 是因为配置变化而导致的销毁时，不会将 `ViewModelStore` 清理掉。

##### 2、FragmentManagerViewModel 的 clearNonConfigState 方法中被调用

```java
class FragmentManagerViewModel extends ViewModel {
  	...
  	void clearNonConfigState(@NonNull Fragment f) {
        ...
        // Clear and remove the Fragment's ViewModelStore
        ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
        if (viewModelStore != null) {
            viewModelStore.clear();
            mViewModelStores.remove(f.mWho);
        }
    }
  	...
}
```

此处用于清理 `Fragment` 中的 `ViewModelStore`。而该方法在 `FragmentManagerImpl` 的 `moveToState` 方法中会被调用：

```java
boolean beingRemoved = f.mRemoving && !f.isInBackStack();
if (beingRemoved || mNonConfig.shouldDestroy(f)) {
    boolean shouldClear;
    if (mHost instanceof ViewModelStoreOwner) {
        shouldClear = mNonConfig.isCleared();
    } else if (mHost.getContext() instanceof Activity) {
        Activity activity = (Activity) mHost.getContext();
        shouldClear = !activity.isChangingConfigurations();
    } else {
        shouldClear = true;
    }
    if (beingRemoved || shouldClear) {
      	// 清除 ViewModelStore
        mNonConfig.clearNonConfigState(f);
    }
    f.performDestroy();
    dispatchOnFragmentDestroyed(f, false);
} else {
    ...
}
```

当 `Fragment` 的状态发生改变时会执行 `moveToState` 方法。而当 `Fragment` 被移除或者应该被清除时执行 `FragmentManagerViewModel` 的 `clearNonConfigState` 方法。



### ViewModelStoreOwner

```java
public interface ViewModelStoreOwner {
    /**
     * Returns owned {@link ViewModelStore}
     *
     * @return a {@code ViewModelStore}
     */
    @NonNull
    ViewModelStore getViewModelStore();
}
```

`ViewModelStoreOwner` 接口定义了获取 `ViewModelStore` 的方法。`ComponentActivity` 和 `Fragment` 都实现了此接口。

##### 1、ComponentActivity 中的 `getViewModelStore`

```java
@NonNull
@Override
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}
```

首先尝试通过 `NonConfigurationInstances` 恢复 `ViewModelStore`，如果为空则创建新的对象。

##### 2、Fragment 中的 `getViewModelStore`

```java
// The fragment manager we are associated with.  Set as soon as the
// fragment is used in a transaction; cleared after it has been removed
// from all transactions.
FragmentManagerImpl mFragmentManager;

@NonNull
@Override
public ViewModelStore getViewModelStore() {
    if (mFragmentManager == null) {
        throw new IllegalStateException("Can't access ViewModels from detached fragment");
    }
    return mFragmentManager.getViewModelStore(this);
}
```

在 `Fragment` 的 `getViewModelStore` 方法中调用了 `FragmentManagerImpl` 的 `getViewModelStore` 方法，并传入自身当参数：

```java
final class FragmentManagerImpl extends FragmentManager implements LayoutInflater.Factory2 {
  private FragmentManagerViewModel mNonConfig;

  @NonNull
  ViewModelStore getViewModelStore(@NonNull Fragment f) {
      return mNonConfig.getViewModelStore(f);
  }
}
```

其中调用了 `FragmentManagerViewModel`  的 `getViewModelStore` 方法。

```java
class FragmentManagerViewModel extends ViewModel {
  	@NonNull
    ViewModelStore getViewModelStore(@NonNull Fragment f) {
        ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
        if (viewModelStore == null) {
            viewModelStore = new ViewModelStore();
            mViewModelStores.put(f.mWho, viewModelStore);
        }
        return viewModelStore;
    }
}
```

同样的，首先尝试从 `mViewModelStores` 中获取，如果为空则创建新的对象并存入 `mViewModleStores` 当中。



#### Activity 中的 ViewModelStore 的重建保留

上文可以看到在 `getViewModelStore` 方法中首先调用了 `getLastNonConfigurationInstace` 方法尝试恢复 `ViewModelStore`。

```java
static final class NonConfigurationInstances {
    Object activity;
    HashMap<String, Object> children;
    FragmentManagerNonConfig fragments;
    ArrayMap<String, LoaderManager> loaders;
    VoiceInteractor voiceInteractor;
}
@UnsupportedAppUsage
/* package */ NonConfigurationInstances mLastNonConfigurationInstances;

@Nullable
public Object getLastNonConfigurationInstance() {
    return mLastNonConfigurationInstances != null
      			? mLastNonConfigurationInstances.activity : null;
}
```

当 `mLastNonConfigurationInstances` 不为空时则返回它的 `Object` 类型的变量 `activity`。那么，这个 `mLastNonConfigurationInstances` 又是在何时被赋值的呢？答案就在 `attach` 方法中：

```java
@UnsupportedAppUsage
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
  	...
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
  	..
}
```

在 `ActivityThread` 中，`performLaunchActivity` 方法中会执行 `attach` 方法。

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  	...
    	activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);
  	...
}
```

那么 `ActivityClientRecord` 的 `lastNonConfigurationInstances` 又是在何时被赋值的呢？那就是在 `performDestroyActivity` 方法中执行了 `retainNonConfigurationInstaces` 方法并将其返回值赋值给了 `ActivityClientRecord` 的 `lastNonConfigurationInstances`。

```java


private void handleRelaunchActivityInner(ActivityClientRecord r, int configChanges,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingIntents,
            PendingTransactionActions pendingActions, boolean startsNotResumed,
            Configuration overrideConfig, String reason) {
  	...
    handleDestroyActivity(r.token, false, configChanges, true, reason);
  	...
}

ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
        int configChanges, boolean getNonConfigInstance, String reason) {
  	...
      if (getNonConfigInstance) {
        try {
            r.lastNonConfigurationInstances
              	= r.activity.retainNonConfigurationInstances();
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException(
                  "Unable to retain activity "
                  + r.intent.getComponent().toShortString()
                  + ": " + e.toString(), e);
            }
        }
      }
  	...
}
```

在 `Activity` 的 `retainNonConfigurationInstances` 方法中：

```java
NonConfigurationInstances retainNonConfigurationInstances() {
    Object activity = onRetainNonConfigurationInstance();
    HashMap<String, Object> children = onRetainNonConfigurationChildInstances();
    FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();

    // We're already stopped but we've been asked to retain.
    // Our fragments are taken care of but we need to mark the loaders for retention.
    // In order to do this correctly we need to restart the loaders first before
    // handing them off to the next activity.
    mFragments.doLoaderStart();
    mFragments.doLoaderStop(true);
    ArrayMap<String, LoaderManager> loaders = mFragments.retainLoaderNonConfig();

    if (activity == null && children == null && fragments == null && loaders == null
            && mVoiceInteractor == null) {
        return null;
    }

    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.activity = activity;
    nci.children = children;
    nci.fragments = fragments;
    nci.loaders = loaders;
    if (mVoiceInteractor != null) {
        mVoiceInteractor.retainInstance();
        nci.voiceInteractor = mVoiceInteractor;
    }
    return nci;
}
```

其中首先调用了 `onRetainNonConfigurationInstance` 方法，该方法在 `ComponentActivity` 中被实现。

```java
@Override
@Nullable
public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();

    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        // No one called getViewModelStore(), so see if there was an existing
        // ViewModelStore from our last NonConfigurationInstance
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            viewModelStore = nc.viewModelStore;
        }
    }

    if (viewModelStore == null && custom == null) {
        return null;
    }

    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = viewModelStore;
    return nci;
}
```

在这里创建了 `NonConfigurationInstances` 并将 `viewModelStore` 赋值给了它的 `viewModelStore` 变量。



### Fragment 中的保留

```java
/**
 * Control whether a fragment instance is retained across Activity
 * re-creation (such as from a configuration change). If set, the fragment
 * lifecycle will be slightly different when an activity is recreated:
 * <ul>
 * <li> {@link #onDestroy()} will not be called (but {@link #onDetach()} still
 * will be, because the fragment is being detached from its current activity).
 * <li> {@link #onCreate(Bundle)} will not be called since the fragment
 * is not being re-created.
 * <li> {@link #onAttach(Activity)} and {@link #onActivityCreated(Bundle)} <b>will</b>
 * still be called.
 * </ul>
 */
public void setRetainInstance(boolean retain) {
    mRetainInstance = retain;
    if (mFragmentManager != null) {
        if (retain) {
            mFragmentManager.addRetainedFragment(this);
        } else {
            mFragmentManager.removeRetainedFragment(this);
        }
    } else {
        mRetainInstanceChangedWhileDetached = true;
    }
}
```

将 `Fragment` 设置为 retain 状态，在 `Activity` 重建时 `Fragment` 将不会被销毁重建，而是保留。