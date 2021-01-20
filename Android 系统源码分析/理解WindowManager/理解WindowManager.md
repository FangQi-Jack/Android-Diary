# 理解 WindowManager

## Window、WindowManager 和 WindowManagerService
![Window 、 WindowManager 和 WMS](https://i.imgur.com/o1mtVZZ.png)

Window 是一个抽象类，它的具体实现类是 PhoneWindow ，它对 View 进行管理。 WindowManager 是一个接口，继承自 ViewManager 接口，用于管理 Window ，它的具体实现类是 WindowManagerImpl 。	通过 WindowManager 可以对 Window 进行添加、更新和删除的操作，而 WindowManager 会将具体的工作交给 WMS（ WindowMangerService）去完成， WindowManager 和 WMS 之间通过 Binder 进行跨进程通信。 WindowManager 和 WMS 的关系类似于 ActivityManager 和 AMS 。

## WindowManager 的关联类
![WindowManager 的关联类](https://i.imgur.com/zfooEMb.png)

WindowManager 继承自 ViewManager ， ViewManager 中定义了三个方法，分别是添加、更新和删除 View ：

```
public interface ViewManager
{
    /**
     * Assign the passed LayoutParams to the passed View and add the view to the window.
     * <p>Throws {@link android.view.WindowManager.BadTokenException} for certain programming
     * errors, such as adding a second view to a window without removing the first view.
     * <p>Throws {@link android.view.WindowManager.InvalidDisplayException} if the window is on a
     * secondary {@link Display} and the specified display can't be found
     * (see {@link android.app.Presentation}).
     * @param view The view to be added to this window.
     * @param params The LayoutParams to assign to view.
     */
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}

```
可以看到这些方法的参数都是 View 类型的，说明了 Window 是以 View 的形式存在的。 WindowManager 在继承 ViewManager 的同时也加入了很多其他功能，包括 Window 的类型和层级相关的常量、内部类和一些方法，其中两个方法是根据 Window 的特性而加入的：

```
/**
 * Returns the {@link Display} upon which this {@link WindowManager} instance
 * will create new windows.
 * <p>
 * Despite the name of this method, the display that is returned is not
 * necessarily the primary display of the system (see {@link Display#DEFAULT_DISPLAY}).
 * The returned display could instead be a secondary display that this
 * window manager instance is managing.  Think of it as the display that
 * this {@link WindowManager} instance uses by default.
 * </p><p>
 * To create windows on a different display, you need to obtain a
 * {@link WindowManager} for that {@link Display}.  (See the {@link WindowManager}
 * class documentation for more information.)
 * </p>
 *
 * @return The display that this window manager is managing.
 */
public Display getDefaultDisplay();

/**
 * Special variation of {@link #removeView} that immediately invokes
 * the given view hierarchy's {@link View#onDetachedFromWindow()
 * View.onDetachedFromWindow()} methods before returning.  This is not
 * for normal applications; using it correctly requires great care.
 *
 * @param view The view to be removed.
 */
public void removeViewImmediate(View view);

```
getDefaultDisplay 方法可以获取到 WindowManager 管理的 display（屏幕）。 removeViewImmediate 方法规定了在这个方法返回前要立即执行 View 的 onDetachedFromWindow 方法，来完成对传入的 View 的销毁。

Window 是一个抽象类，它的具体实现类是 PhoneWindow ，那么 PhoneWindow 又是在何时创建的呢？答案是在 Activity 的启动过程中创建的。 Activity 的启动过程会调用 ActivityThread 的 performLaunchActivity 方法，在该方法中会调用 Activity 的 attach 方法，正是在 attach 方法中完成了 PhoneWindow 的创建工作。如下所示（已省略其他不相关代码）：

```
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback) {
    ...

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    
    ...

    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    
    ...
}

```
从源码中可以看到，创建了 PhoneWindow ，并且调用了 setWindowManager 方法。而 setWindowManager 方法是在 Window 中实现的：

```
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated
            || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}

```
如果传入的 WindowManager 为 null ，则会调用 Context 的 getSystemService 方法，并传入 Context.WINDOW_SERVICE 作为参数，具体在 ContextImpl 中实现：

```
@Override
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}

在 SystemServiceRegistry 中：

public static Object getSystemService(ContextImpl ctx, String name) {
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}

```
SYSTEM_SERVICE_FETCHERS 是一个 HashMap ，保存了服务的 fetcher，在 SystemServiceRegistry 的 static 代码块中会调用 registerService 方法注册各种服务： 

```
final class SystemServiceRegistry {
    private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
        new HashMap<String, ServiceFetcher<?>>();

    // Not instantiable.
    private SystemServiceRegistry() { }

    static {
        ...

        registerService(Context.WINDOW_SERVICE, WindowManager.class,
            new CachedServiceFetcher<WindowManager>() {
        @Override
        public WindowManager createService(ContextImpl ctx) {
            return new WindowManagerImpl(ctx);
        }});

        ...
    }

    private static <T> void registerService(String serviceName, Class<T> serviceClass,
        ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }

    ...

    static abstract class CachedServiceFetcher<T> implements ServiceFetcher<T> {
        private final int mCacheIndex;

        public CachedServiceFetcher() {
            mCacheIndex = sServiceCacheSize++;
        }

        @Override
        @SuppressWarnings("unchecked")
        public final T getService(ContextImpl ctx) {
            final Object[] cache = ctx.mServiceCache;
            synchronized (cache) {
                // Fetch or create the service.
                Object service = cache[mCacheIndex];
                if (service == null) {
                    try {
                        service = createService(ctx);
                        cache[mCacheIndex] = service;
                    } catch (ServiceNotFoundException e) {
                        onServiceNotFound(e);
                    }
                }
                return (T)service;
            }
        }

        public abstract T createService(ContextImpl ctx) throws ServiceNotFoundException;
    }

    ...    
}

```
可以看到 Context.WINDOW_SERVICE 对应的就是 WindowManagerImpl 的实例，因此 Context 的 getSystemService 方法得到的就是 WindowManagerImpl 的实例。再回到 Window 的 setWindowManager 方法，在得到 WindowManagerImpl 实例后转为 WindowManager ，然后调用了 WindowManagerImpl 的 createLocalWindowManager 方法：

```
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}

```
该方法也同样是创建了 WindowManagerImpl ，不同的是这时候会将 Window 作为参数传入，这样 WindowManagerImpl 就持有了 Window 的引用：

```
public final class WindowManagerImpl implements WindowManager {
    ...

    private final Window mParentWindow;

    ...

    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

    ...
}

```
如此就可以对 Window 进行操作了，比如在 Window 中添加 View ，则会调用 WindowManagerImpl 的 addView 方法：

```
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    
    private final Window mParentWindow;
    ...

    private WindowManagerImpl(Context context, Window parentWindow) {
        ...
        mParentWindow = parentWindow;
    }

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
}

```
可以看到，虽然 WindowManagerImpl 是 WindowManager 的实现类，但是却并没有实现功能，而是将功能实现委托给了 WindowManagerGlobal，这里用的是**桥接模式**。而 WindowManagerGlobal 是采用单例模式实现的，说明在一个进程中只有一个 WindowManager 实例，一个进程中可能会有多个 WindowManagerImpl。


## Window 的属性
WMS 是 Window 的最终管理者，而为了方便管理则需要定义一些属性，它们就被定义在 WindowManager 的静态内部类 LayoutParams 中。 Window 的属性有很多种，与应用开发最密切的有三种： Type （ Window 类型）、 Flag （ Window 标志）、 SoftInputMode （软键盘相关模式）。

### Window 的类型和显示次序
Window 有很多种类型，比如应用程序窗口、系统错误窗口、输入法窗口、 PopupWindow 、 Toast 、 Dialog 等。总体来说 Window 分为三类： Application Window （应用程序窗口）、 Sub Window （子窗口）、 System Window （系统窗口）。每个大类型中又分为多种类型，都定义在 WindowManager 的 LayoutParams 中。

#### 1、 Application Window
Activity 就是一个典型的 Application Window。 Application Window 包含如下类型：

```
    /**
     * Start of window types that represent normal application windows.
     */
    public static final int FIRST_APPLICATION_WINDOW = 1;

    /**
     * Window type: an application window that serves as the "base" window
     * of the overall application; all other application windows will
     * appear on top of it.
     * In multiuser systems shows only on the owning user's window.
     */
    public static final int TYPE_BASE_APPLICATION   = 1;

    /**
     * Window type: a normal application window.  The {@link #token} must be
     * an Activity token identifying who the window belongs to.
     * In multiuser systems shows only on the owning user's window.
     */
    public static final int TYPE_APPLICATION        = 2;

    /**
     * Window type: special application window that is displayed while the
     * application is starting.  Not for use by applications themselves;
     * this is used by the system to display something until the
     * application can show its own windows.
     * In multiuser systems shows on all users' windows.
     */
    public static final int TYPE_APPLICATION_STARTING = 3;

    /**
     * Window type: a variation on TYPE_APPLICATION that ensures the window
     * manager will wait for this window to be drawn before the app is shown.
     * In multiuser systems shows only on the owning user's window.
     */
    public static final int TYPE_DRAWN_APPLICATION = 4;

    /**
     * End of types of application windows.
     */
    public static final int LAST_APPLICATION_WINDOW = 99;

```
FIRST_APPLICATION_WINDOW 表示 Application Window 类型的初始值， LAST_APPLICATION_WINDOW 表示 Application Window 类型的结束值，所以 Application Window 的 Type 取值范围为 1 ~ 99 。

#### 2、 Sub Window
子窗口是不能独立存在的，需要依附于其他类型的窗口， PopupWindow 就属于 Sub Window。 Sub Window 的类型包含：

```
    /**
     * Start of types of sub-windows.  The {@link #token} of these windows
     * must be set to the window they are attached to.  These types of
     * windows are kept next to their attached window in Z-order, and their
     * coordinate space is relative to their attached window.
     */
    public static final int FIRST_SUB_WINDOW = 1000;

    /**
     * Window type: a panel on top of an application window.  These windows
     * appear on top of their attached window.
     */
    public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;

    /**
     * Window type: window for showing media (such as video).  These windows
     * are displayed behind their attached window.
     */
    public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;

    /**
     * Window type: a sub-panel on top of an application window.  These
     * windows are displayed on top their attached window and any
     * {@link #TYPE_APPLICATION_PANEL} panels.
     */
    public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;

    /** Window type: like {@link #TYPE_APPLICATION_PANEL}, but layout
     * of the window happens as that of a top-level window, <em>not</em>
     * as a child of its container.
     */
    public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;

    /**
     * Window type: window for showing overlays on top of media windows.
     * These windows are displayed between TYPE_APPLICATION_MEDIA and the
     * application window.  They should be translucent to be useful.  This
     * is a big ugly hack so:
     * @hide
     */
    public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4;

    /**
     * Window type: a above sub-panel on top of an application window and it's
     * sub-panel windows. These windows are displayed on top of their attached window
     * and any {@link #TYPE_APPLICATION_SUB_PANEL} panels.
     * @hide
     */
    public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;

    /**
     * End of types of sub-windows.
     */
    public static final int LAST_SUB_WINDOW = 1999;

```
因此 Sub Window 的 Type 值在 1000 ~ 1999 之间。

#### 3、 System Window
System Window 有很多种类型，例如 Toast 、 输入法窗口、系统音量条窗口等，它的类型定义如下：

```
    /**
     * Start of system-specific window types.  These are not normally
     * created by applications.
     */
    public static final int FIRST_SYSTEM_WINDOW     = 2000;

    /**
     * Window type: the status bar.  There can be only one status bar
     * window; it is placed at the top of the screen, and all other
     * windows are shifted down so they are below it.
     * In multiuser systems shows on all users' windows.
     */
    public static final int TYPE_STATUS_BAR         = FIRST_SYSTEM_WINDOW;

    /**
     * Window type: the search bar.  There can be only one search bar
     * window; it is placed at the top of the screen.
     * In multiuser systems shows on all users' windows.
     */
    public static final int TYPE_SEARCH_BAR         = FIRST_SYSTEM_WINDOW+1;

    /**
     * Window type: phone.  These are non-application windows providing
     * user interaction with the phone (in particular incoming calls).
     * These windows are normally placed above all applications, but behind
     * the status bar.
     * In multiuser systems shows on all users' windows.
     * @deprecated for non-system apps. Use {@link #TYPE_APPLICATION_OVERLAY} instead.
     */
    @Deprecated
    public static final int TYPE_PHONE              = FIRST_SYSTEM_WINDOW+2;

    /**
     * Window type: system window, such as low power alert. These windows
     * are always on top of application windows.
     * In multiuser systems shows only on the owning user's window.
     * @deprecated for non-system apps. Use {@link #TYPE_APPLICATION_OVERLAY} instead.
     */
    @Deprecated
    public static final int TYPE_SYSTEM_ALERT       = FIRST_SYSTEM_WINDOW+3;

    /**
     * Window type: keyguard window.
     * In multiuser systems shows on all users' windows.
     * @removed
     */
    public static final int TYPE_KEYGUARD           = FIRST_SYSTEM_WINDOW+4;

    /**
     * Window type: transient notifications.
     * In multiuser systems shows only on the owning user's window.
     * @deprecated for non-system apps. Use {@link #TYPE_APPLICATION_OVERLAY} instead.
     */
    @Deprecated
    public static final int TYPE_TOAST              = FIRST_SYSTEM_WINDOW+5;

    /**
     * Window type: system overlay windows, which need to be displayed
     * on top of everything else.  These windows must not take input
     * focus, or they will interfere with the keyguard.
     * In multiuser systems shows only on the owning user's window.
     * @deprecated for non-system apps. Use {@link #TYPE_APPLICATION_OVERLAY} instead.
     */
    @Deprecated
    public static final int TYPE_SYSTEM_OVERLAY     = FIRST_SYSTEM_WINDOW+6;

    /**
     * Window type: priority phone UI, which needs to be displayed even if
     * the keyguard is active.  These windows must not take input
     * focus, or they will interfere with the keyguard.
     * In multiuser systems shows on all users' windows.
     * @deprecated for non-system apps. Use {@link #TYPE_APPLICATION_OVERLAY} instead.
     */
    @Deprecated
    public static final int TYPE_PRIORITY_PHONE     = FIRST_SYSTEM_WINDOW+7;

    /**
     * Window type: panel that slides out from the status bar
     * In multiuser systems shows on all users' windows.
     */
    public static final int TYPE_SYSTEM_DIALOG      = FIRST_SYSTEM_WINDOW+8;

    /**
     * Window type: dialogs that the keyguard shows
     * In multiuser systems shows on all users' windows.
     */
    public static final int TYPE_KEYGUARD_DIALOG    = FIRST_SYSTEM_WINDOW+9;

    /**
     * Window type: internal system error windows, appear on top of
     * everything they can.
     * In multiuser systems shows only on the owning user's window.
     * @deprecated for non-system apps. Use {@link #TYPE_APPLICATION_OVERLAY} instead.
     */
    @Deprecated
    public static final int TYPE_SYSTEM_ERROR       = FIRST_SYSTEM_WINDOW+10;

    /**
     * Window type: internal input methods windows, which appear above
     * the normal UI.  Application windows may be resized or panned to keep
     * the input focus visible while this window is displayed.
     * In multiuser systems shows only on the owning user's window.
     */
    public static final int TYPE_INPUT_METHOD       = FIRST_SYSTEM_WINDOW+11;

    /**
     * Window type: internal input methods dialog windows, which appear above
     * the current input method window.
     * In multiuser systems shows only on the owning user's window.
     */
    public static final int TYPE_INPUT_METHOD_DIALOG= FIRST_SYSTEM_WINDOW+12;

    /**
     * Window type: wallpaper window, placed behind any window that wants
     * to sit on top of the wallpaper.
     * In multiuser systems shows only on the owning user's window.
     */
    public static final int TYPE_WALLPAPER          = FIRST_SYSTEM_WINDOW+13;

    /**
     * Window type: panel that slides out from over the status bar
     * In multiuser systems shows on all users' windows.
     */
    public static final int TYPE_STATUS_BAR_PANEL   = FIRST_SYSTEM_WINDOW+14;

    /**
     * Window type: secure system overlay windows, which need to be displayed
     * on top of everything else.  These windows must not take input
     * focus, or they will interfere with the keyguard.
     *
     * This is exactly like {@link #TYPE_SYSTEM_OVERLAY} except that only the
     * system itself is allowed to create these overlays.  Applications cannot
     * obtain permission to create secure system overlays.
     *
     * In multiuser systems shows only on the owning user's window.
     * @hide
     */
    public static final int TYPE_SECURE_SYSTEM_OVERLAY = FIRST_SYSTEM_WINDOW+15;

    /**
     * Window type: the drag-and-drop pseudowindow.  There is only one
     * drag layer (at most), and it is placed on top of all other windows.
     * In multiuser systems shows only on the owning user's window.
     * @hide
     */
    public static final int TYPE_DRAG               = FIRST_SYSTEM_WINDOW+16;

    /**
     * Window type: panel that slides out from under the status bar
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_STATUS_BAR_SUB_PANEL = FIRST_SYSTEM_WINDOW+17;

    /**
     * Window type: (mouse) pointer
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_POINTER = FIRST_SYSTEM_WINDOW+18;

    /**
     * Window type: Navigation bar (when distinct from status bar)
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_NAVIGATION_BAR = FIRST_SYSTEM_WINDOW+19;

    /**
     * Window type: The volume level overlay/dialog shown when the user
     * changes the system volume.
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_VOLUME_OVERLAY = FIRST_SYSTEM_WINDOW+20;

    /**
     * Window type: The boot progress dialog, goes on top of everything
     * in the world.
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_BOOT_PROGRESS = FIRST_SYSTEM_WINDOW+21;

    /**
     * Window type to consume input events when the systemUI bars are hidden.
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_INPUT_CONSUMER = FIRST_SYSTEM_WINDOW+22;

    /**
     * Window type: Dreams (screen saver) window, just above keyguard.
     * In multiuser systems shows only on the owning user's window.
     * @hide
     */
    public static final int TYPE_DREAM = FIRST_SYSTEM_WINDOW+23;

    /**
     * Window type: Navigation bar panel (when navigation bar is distinct from status bar)
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_NAVIGATION_BAR_PANEL = FIRST_SYSTEM_WINDOW+24;

    /**
     * Window type: Display overlay window.  Used to simulate secondary display devices.
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_DISPLAY_OVERLAY = FIRST_SYSTEM_WINDOW+26;

    /**
     * Window type: Magnification overlay window. Used to highlight the magnified
     * portion of a display when accessibility magnification is enabled.
     * In multiuser systems shows on all users' windows.
     * @hide
     */
    public static final int TYPE_MAGNIFICATION_OVERLAY = FIRST_SYSTEM_WINDOW+27;

    /**
     * Window type: Window for Presentation on top of private
     * virtual display.
     */
    public static final int TYPE_PRIVATE_PRESENTATION = FIRST_SYSTEM_WINDOW+30;

    /**
     * Window type: Windows in the voice interaction layer.
     * @hide
     */
    public static final int TYPE_VOICE_INTERACTION = FIRST_SYSTEM_WINDOW+31;

    /**
     * Window type: Windows that are overlaid <em>only</em> by a connected {@link
     * android.accessibilityservice.AccessibilityService} for interception of
     * user interactions without changing the windows an accessibility service
     * can introspect. In particular, an accessibility service can introspect
     * only windows that a sighted user can interact with which is they can touch
     * these windows or can type into these windows. For example, if there
     * is a full screen accessibility overlay that is touchable, the windows
     * below it will be introspectable by an accessibility service even though
     * they are covered by a touchable window.
     */
    public static final int TYPE_ACCESSIBILITY_OVERLAY = FIRST_SYSTEM_WINDOW+32;

    /**
     * Window type: Starting window for voice interaction layer.
     * @hide
     */
    public static final int TYPE_VOICE_INTERACTION_STARTING = FIRST_SYSTEM_WINDOW+33;

    /**
     * Window for displaying a handle used for resizing docked stacks. This window is owned
     * by the system process.
     * @hide
     */
    public static final int TYPE_DOCK_DIVIDER = FIRST_SYSTEM_WINDOW+34;

    /**
     * Window type: like {@link #TYPE_APPLICATION_ATTACHED_DIALOG}, but used
     * by Quick Settings Tiles.
     * @hide
     */
    public static final int TYPE_QS_DIALOG = FIRST_SYSTEM_WINDOW+35;

    /**
     * Window type: shares similar characteristics with {@link #TYPE_DREAM}. The layer is
     * reserved for screenshot region selection. These windows must not take input focus.
     * @hide
     */
    public static final int TYPE_SCREENSHOT = FIRST_SYSTEM_WINDOW + 36;

    /**
     * Window type: Window for Presentation on an external display.
     * @see android.app.Presentation
     * @hide
     */
    public static final int TYPE_PRESENTATION = FIRST_SYSTEM_WINDOW + 37;

    /**
     * Window type: Application overlay windows are displayed above all activity windows
     * (types between {@link #FIRST_APPLICATION_WINDOW} and {@link #LAST_APPLICATION_WINDOW})
     * but below critical system windows like the status bar or IME.
     * <p>
     * The system may change the position, size, or visibility of these windows at anytime
     * to reduce visual clutter to the user and also manage resources.
     * <p>
     * Requires {@link android.Manifest.permission#SYSTEM_ALERT_WINDOW} permission.
     * <p>
     * The system will adjust the importance of processes with this window type to reduce the
     * chance of the low-memory-killer killing them.
     * <p>
     * In multi-user systems shows only on the owning user's screen.
     */
    public static final int TYPE_APPLICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 38;

    /**
     * End of types of system windows.
     */
    public static final int LAST_SYSTEM_WINDOW      = 2999;

```
可以看到 System Window 的 Type 取值范围是 2000 ~ 2999 。

#### 4、窗口显示次序
当一个进程向 WMS 申请一个窗口时， WMS 会确定该窗口的显示次序。为了方便对窗口显示次序的管理，手机屏幕可以虚拟的用 X 、 Y 、 Z 轴来表示，其中 Z 轴垂直于屏幕，从屏幕内向屏幕外，因此确定窗口显示次序也就是确定窗口在 Z 轴上的次序，这个次序称为 Z-Order 。 Type 值是确定 Z-Order 的依据。一般情况下， Type 值越大 Z-Order 排序越靠前，即越靠近用户。当然窗口显示次序的逻辑不会这么简单，当多个窗口的 Type 值相同时， WMS 会结合各种情况来最终确定 Z-Order。

### Window 的标志
Window 的标志也就是 Flag，它控制着 Window 的显示。在 WindowManager 的 LayoutParams 中总共定义了 20 多种 Flag 。设置 Window 的 Flag 有 3 中方式：

#### 1、通过 Window 的 addFlags 方法

```
Window window = getWindow();
window.addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);

```
#### 2、通过 Window 的 setFlags 方法

```
Window window = getWindow();
window.addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);

```
addFlags 方法内部会调用 setFlags ，因此二者区别不大。

#### 3、通过给 LayoutParams 设置 Flag

```
WindowManager.LayoutParams lp = new WindowManager.LayoutParams();
lp.flags = WindowManager.LayoutParams.FLAG_FULLSCREEN;
TextView tv = new TextView(this);
WindowManager wm = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
wm.addView(tv, lp);

```
### SoftInputMode
窗口与窗口叠加的场景非常常见，然而当其中的窗口是软键盘窗口时，则可能会带来一些问题，比如软键盘遮挡了输入框。为了使软键盘窗口能按照期望的显示，在 WindowManager 的 LayoutParams 中定义了软键盘相关模式。

```
/**
     * Mask for {@link #softInputMode} of the bits that determine the
     * desired visibility state of the soft input area for this window.
     */
    public static final int SOFT_INPUT_MASK_STATE = 0x0f;

    /**
     * Visibility state for {@link #softInputMode}: no state has been specified.
     */
    public static final int SOFT_INPUT_STATE_UNSPECIFIED = 0;

    /**
     * Visibility state for {@link #softInputMode}: please don't change the state of
     * the soft input area.
     */
    public static final int SOFT_INPUT_STATE_UNCHANGED = 1;

    /**
     * Visibility state for {@link #softInputMode}: please hide any soft input
     * area when normally appropriate (when the user is navigating
     * forward to your window).
     */
    public static final int SOFT_INPUT_STATE_HIDDEN = 2;

    /**
     * Visibility state for {@link #softInputMode}: please always hide any
     * soft input area when this window receives focus.
     */
    public static final int SOFT_INPUT_STATE_ALWAYS_HIDDEN = 3;

    /**
     * Visibility state for {@link #softInputMode}: please show the soft
     * input area when normally appropriate (when the user is navigating
     * forward to your window).
     */
    public static final int SOFT_INPUT_STATE_VISIBLE = 4;

    /**
     * Visibility state for {@link #softInputMode}: please always make the
     * soft input area visible when this window receives input focus.
     */
    public static final int SOFT_INPUT_STATE_ALWAYS_VISIBLE = 5;

    /**
     * Mask for {@link #softInputMode} of the bits that determine the
     * way that the window should be adjusted to accommodate the soft
     * input window.
     */
    public static final int SOFT_INPUT_MASK_ADJUST = 0xf0;

    /** Adjustment option for {@link #softInputMode}: nothing specified.
     * The system will try to pick one or
     * the other depending on the contents of the window.
     */
    public static final int SOFT_INPUT_ADJUST_UNSPECIFIED = 0x00;

    /** Adjustment option for {@link #softInputMode}: set to allow the
     * window to be resized when an input
     * method is shown, so that its contents are not covered by the input
     * method.  This can <em>not</em> be combined with
     * {@link #SOFT_INPUT_ADJUST_PAN}; if
     * neither of these are set, then the system will try to pick one or
     * the other depending on the contents of the window. If the window's
     * layout parameter flags include {@link #FLAG_FULLSCREEN}, this
     * value for {@link #softInputMode} will be ignored; the window will
     * not resize, but will stay fullscreen.
     */
    public static final int SOFT_INPUT_ADJUST_RESIZE = 0x10;

    /** Adjustment option for {@link #softInputMode}: set to have a window
     * pan when an input method is
     * shown, so it doesn't need to deal with resizing but just panned
     * by the framework to ensure the current input focus is visible.  This
     * can <em>not</em> be combined with {@link #SOFT_INPUT_ADJUST_RESIZE}; if
     * neither of these are set, then the system will try to pick one or
     * the other depending on the contents of the window.
     */
    public static final int SOFT_INPUT_ADJUST_PAN = 0x20;

    /** Adjustment option for {@link #softInputMode}: set to have a window
     * not adjust for a shown input method.  The window will not be resized,
     * and it will not be panned to make its focus visible.
     */
    public static final int SOFT_INPUT_ADJUST_NOTHING = 0x30;

    /**
     * Bit for {@link #softInputMode}: set when the user has navigated
     * forward to the window.  This is normally set automatically for
     * you by the system, though you may want to set it in certain cases
     * when you are displaying a window yourself.  This flag will always
     * be cleared automatically after the window is displayed.
     */
    public static final int SOFT_INPUT_IS_FORWARD_NAVIGATION = 0x100;

```
除了在 Manifest 中为 Activity 设置 android:softInputMode 外还可以通过代码的方式为 Window 设置 softInputMode ：

```
getWindow().setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);

```
## Window 的操作
![Window 的操作](https://i.imgur.com/3npyflA.png)

对于 Window 的操作，最终都是由 WMS 来完成的，Window 的操作分为两大部分： WindowManager 的操作部分和 WMS 的操作部分。

### System Window 的添加过程
System Window 有很多种类型，不同类型的 System Window 的添加过程也不尽相同，我们以 StatusBar 为例，StatusBar 是 SystemUI 的重要组成部分。在 StatusBar 中，addStatusBarWindow 这个方法负责为 StatusBar 添加 Window ：

```
private void addStatusBarWindow() {
    makeStatusBarView();
    mStatusBarWindowManager = Dependency.get(StatusBarWindowManager.class);
    mRemoteInputController = new RemoteInputController(mHeadsUpManager);
    mStatusBarWindowManager.add(mStatusBarWindow, getStatusBarHeight());
}

```
可以看到，在这个方法中，首先会调用 makeStatusBarView 来构建 StatusBar ，然后会获取 StatusBarWindowManager ，并调用它的 add 方法：

```
public void add(View statusBarView, int barHeight) {

    // Now that the status bar window encompasses the sliding panel and its
    // translucent backdrop, the entire thing is made TRANSLUCENT and is
    // hardware-accelerated.
    mLp = new WindowManager.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            barHeight,
            WindowManager.LayoutParams.TYPE_STATUS_BAR,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING
                    | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
                    | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
                    | WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
            PixelFormat.TRANSLUCENT);
    mLp.token = new Binder();
    mLp.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
    mLp.gravity = Gravity.TOP;
    mLp.softInputMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
    mLp.setTitle("StatusBar");
    mLp.packageName = mContext.getPackageName();
    mStatusBarView = statusBarView;
    mBarHeight = barHeight;
    mWindowManager.addView(mStatusBarView, mLp);
    mLpChanged = new WindowManager.LayoutParams();
    mLpChanged.copyFrom(mLp);
}

```
在 add 方法中首先会创建 LayoutParams 来为 StatusBar 配置属性，其中就将 StatusBar 的 type 设置成了 WindowManager.LayoutParams.TYPE_STATUS_BAR ，说明 StatusBar 的窗口类型为状态栏。然后又调用了 WindowManager 的 addView 方法，该方法是 WindowManager 的父类 ViewManager 中的方法，具体的实现是在 WindowManagerImpl 中：

```
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}

```
该方法调用了 WindowManagerGlobal 的 addView 方法：

```
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        // If there's no parent, then hardware acceleration for this view is
        // set from the application's hardware acceleration setting.
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        // Start watching for system property changes.
        if (mSystemPropertyUpdater == null) {
            mSystemPropertyUpdater = new Runnable() {
                @Override public void run() {
                    synchronized (mLock) {
                        for (int i = mRoots.size() - 1; i >= 0; --i) {
                            mRoots.get(i).loadSystemProperties();
                        }
                    }
                }
            };
            SystemProperties.addChangeCallback(mSystemPropertyUpdater);
        }

        int index = findViewLocked(view, false);
        if (index >= 0) {
            if (mDyingViews.contains(view)) {
                // Don't wait for MSG_DIE to make it's way through root's queue.
                mRoots.get(index).doDie();
            } else {
                throw new IllegalStateException("View " + view
                        + " has already been added to the window manager.");
            }
            // The previous removeView() had not completed executing. Now it has.
        }

        // If this is a panel window, then find the window it is being
        // attached to for future reference.
        if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            final int count = mViews.size();
            for (int i = 0; i < count; i++) {
                if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                    panelParentView = mViews.get(i);
                }
            }
        }

        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}

```
在 WindowManagerGlobal 中维护了三个 ArrayList： View 列表、 ViewRoomImpl 列表和 LayoutParams 列表

```
private final ArrayList<View> mViews = new ArrayList<View>();
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.LayoutParams>();

```
在 addView 中首先会检查参数。如果当前窗口要作为 Sub Window，则会调用 adjustLayoutParamsForSubWindow 方法对子窗口的 LayoutParams 进行相应的调整。然后会将 View 保存到 View 列表中，将 LayoutParams 保存到 LayoutParams 列表中，然后创建 ViewRoomImpl 并保存，而且还调用了 ViewRootImpl 的 setView 方法，所以添加窗口是通过 ViewRootImpl 来完成的。 ViewRootImpl 的主要职责有：

* View 树的根并管理 View 树
* 触发 View 的测量、布局和绘制
* 输入事件的中转
* 管理 Surface
* 负责与 WMS 进行通信

ViewRoomImpl 的 setView 方法：

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        ...

        try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(),
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mInputChannel);
            }

        ...
    }
}

```
可以看到，调用了 mWindowSession 的 addToDisplay 方法，mWindowSession 是 IWindowSession 类型的，它是一个 Binder 对象， IWindowSession 是 Client 端， Server 端是 Session，而 Session 的 addToDisplay 方法则运行在 WMS 中。

Session 的 addToDisplay ：

```
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}

```
其中 mService 就是 WindowManagerService。addToDisplay 方法最终调用了 WMS 的 addWindow 方法，并将自身也就是 Session 作为参赛传入，每个应用程序进程都会对应一个 Session。

![ViewRootImpl 与 WMS 的通信](https://i.imgur.com/3tRsS0R.png)

![StatusBar 添加过程时序图](https://i.imgur.com/zyHViVO.png)

### Activity 的添加过程
Activity 要与用户进行交互就会调用 ActivityThread 的 handleResumeActivity 方法，在此方法中会获取 Activity 的 WindowManager ，然后调用 WindowManager 的 addView 方法，而第一个参数就是 DecorView，说明 Activity 窗口中会包含 DecorView。

```
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ...

        final Activity a = r.activity;

        ViewManager wm = a.getWindowManager();

        r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // Normally the ViewRoot sets up callbacks with the Activity
                // in addView->ViewRootImpl#setView. If we are instead reusing
                // the decor view we have to notify the view root that the
                // callbacks may have changed.
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {@link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }

    ...
}

```
### Window 的更新
Window 的更新需要调用 updateViewLayout 方法，而在 WindowManagerImpl 的 updateViewLayout 方法中又会调用 WindowManagerGlobal 的 updateViewLayout 方法：

```
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

    view.setLayoutParams(wparams);

    synchronized (mLock) {
        int index = findViewLocked(view, true);
        ViewRootImpl root = mRoots.get(index);
        mParams.remove(index);
        mParams.add(index, wparams);
        root.setLayoutParams(wparams, false);
    }
}

```
在该方法中，会将新的 LayoutParams 设置给 View ，然后会更新 ViewRootImpl 列表 和 LayoutParams 列表，并将新的 LayoutParams 设置给 ViewRootImpl ，在 ViewRootImpl 的 setLayoutParams 中会调用 scheduleTraversals 方法：

```
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

```
mChoreographer 的 postCallBack 用于添加回调，在下一帧被渲染时执行：

```
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

```
在 run 方法中又调用了 doTraversal 方法：

```
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}

```
调用了 performTraversals 方法， performTraversals 方法使得 ViewTree 开始 View 的绘制流程：

```
private void performTraversals() {
    ...

    relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);

    ...

    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

    ...

    performLayout(lp, mWidth, mHeight);

    ...

    performDraw();
}

```
在该方法中的 relayoutWindow 会调用 IWindowSession 的 relayout 方法，跟 StatusBar 的添加一样，最终会调用 WMS 的 relayoutWindow 方法来完成 Window 的更新。另外还会依次调用 performMeasure 、 performLayout 、 performDraw ，而这三个方法内部又会调用 View 的 measure 、 layout 、 draw ，如此就完成了 View 的绘制流程，更新了 Window 。

	