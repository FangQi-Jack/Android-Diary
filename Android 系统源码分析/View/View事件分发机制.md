## View 的滑动

View 是 Android 中所有控件的基类，是控件的一种抽象。

### MotionEvent

典型的 MotionEvent 事件有：

* ACTION_DOWN
* ACTION_MOVE
* ACTION_UP

通过 MotionEvent 可以获取点击事件的坐标，方法：`getX` 和 `getY`、`getRawX` 和 `getRawY`。其中 `getX`/`getY` 获取的是相对于当前 View 左上角的坐标，而 `getRawX` 和 `getRawY` 获取的是相对于手机屏幕左上角的坐标。



### TouchSlop

这是系统能识别出的被认为是滑动的最小距离，它是一个常量，与设备有关，在不同的设备上可能是不同的值。可以通过以下方法获取：`ViewConfiguration.get(context).getScaledTouchSlop()`。在处理滑动时可以通过该值来判断是否达到滑动距离的临界值。在 `frameworks/base/core/res/res/values/config.xml` 中找到该值的定义。



### VelocityTracker

```java
private static native void nativeClear(long ptr);
private static native void nativeComputeCurrentVelocity(long ptr, int units, float maxVelocity);
private static native float nativeGetXVelocity(long ptr, int id);
private static native float nativeGetYVelocity(long ptr, int id);

static public VelocityTracker obtain() {
  VelocityTracker instance = sPool.acquire();
  return (instance != null) ? instance : new VelocityTracker(null);
}

public void computeCurrentVelocity(int units) {
    nativeComputeCurrentVelocity(mPtr, units, Float.MAX_VALUE);
}

public void computeCurrentVelocity(int units, float maxVelocity) {
  nativeComputeCurrentVelocity(mPtr, units, maxVelocity);
}

public float getXVelocity() {
  return nativeGetXVelocity(mPtr, ACTIVE_POINTER_ID);
}

public float getYVelocity() {
  return nativeGetYVelocity(mPtr, ACTIVE_POINTER_ID);
}

public void clear() {
  nativeClear(mPtr);
}

public void recycle() {
  if (mStrategy == null) {
    clear();
    sPool.release(this);
  }
}
```

如上源码所示，可以通过 `obtain()` 方法获取到 VelocityTracker 对象，然后调用 `computeCurrentVelocity` 方法计算当前滑动速度，其入参代表的是一个时间单元或者时间间隔，单位毫秒，即通过此方法计算得到的速度是在这个时间间隔内在水平或垂直方向上所滑动的像素数。

> ***WARN***：
>
> * **获取速度之前必须先计算速度**。即调用 `getXVelocity()` / `getYVelocity()` 方法前必须先执行 `computeCurrentVelocity` 方法。
> * **速度指的是在一段时间内滑过的像素数。速度可以为负数（逆着 x/y 轴方向滑动）。**
> * **仅在需要时调用 `computeCurrentVelocity` 方法，因为此方法开销昂贵。**
> * **在不需要使用时需要调用 `clear()` 方法重置并回收内存。**



### GestureDetector

用于辅助检测单击、滑动、长按、双击等行为。



### Scroller

弹性滑动对象，用于实现 View 的弹性滑动。需要与 View 的 `computeScroll` 方法配合使用才能完成弹性滑动。通过 `computeScrollOffset` 方法不断的根据时间的流逝来计算当前的 scrollX 和 scrollY 的值，在 View 的 `compteScroll` 方法（该方法在 View 中默认为空实现，需要自己实现）中不断调用 `computeScrollOffset` 方法来更新当前的 scrollX 和 scrollY，并重绘 View，在 View 的 `draw` 方法中又会调用 `computeScroll` 方法，如此循环完成滑动。

```java
public class Scroller  {
  public void startScroll(int startX, int startY, int dx, int dy, int duration) {
    mMode = SCROLL_MODE;
    mFinished = false;
    mDuration = duration;
    mStartTime = AnimationUtils.currentAnimationTimeMillis();
    mStartX = startX;
    mStartY = startY;
    mFinalX = startX + dx;
    mFinalY = startY + dy;
    mDeltaX = dx;
    mDeltaY = dy;
    mDurationReciprocal = 1.0f / (float) mDuration;
  }
  
  public boolean computeScrollOffset() {
    if (mFinished) {
      return false;
    }
    
    int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    
        if (timePassed < mDuration) {
            switch (mMode) {
              case SCROLL_MODE:
                  final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                  mCurrX = mStartX + Math.round(x * mDeltaX);
                  mCurrY = mStartY + Math.round(x * mDeltaY);
                  break;
              ...
            }
          ...
        }
  }
}
```



### View 的滑动

实现 View 滑动的三种方式：

* View 自身提供的 `scrollTo` 和 `scrollBy` 方法。
* 使用动画。
* 改变 LayoutParams 使 View 重新布局。

#### 1、scrollTo/scrollBy

```java
protected int mScrollX;

protected int mScrollY;

public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) {
            postInvalidateOnAnimation();
        }
    }
}

public void scrollBy(int x, int y) {
  scrollTo(mScrollX + x, mScrollY + y);
}

public final int getScrollX() {
  return mScrollX;
}

public final int getScrollY() {
  return mScrollY;
}
```

`scrollBy` 内部也是通过调用 `scrollTo` 方法实现滑动的。二者的区别在于：`scrollBy` 完成基于当前位置的相对滑动，`scrollTo` 实现了基于入参值的绝对滑动。这两个方法只能改变 View 内容的位置而不能改变 View 在布局中的位置，即只能移动 View 的内容，而不能移动 View 本身。`mScrollX`  是 View 内容在水平方向滑动的距离，如果从左向右滑动，`mScrollX` 为负，反之为正。`mScrollY` 是 View 的内容在竖直方向上滑动的距离，如果从上往下滑动 `mScrollY` 为负，反之为正。

#### 2、使用动画

主要通过操作 View 的 `translationX` 和 `translationY` 属性来完成移动，可以使用 View 动画也可以使用属性动画。View 动画只能对 View 的影像进行移动，并不能真正的移动 View，且如果 `fillAfter` 被设置为 false，在动画执行完成的瞬间 View 将回到原来的位置，View 的点击事件也会受到影响，比如 View 移动后，点击事件的响应区域依然在原来的位置。使用属性动画不存在这些问题。

#### 3、改变 LayoutParams

比如可以通过改变 LayoutParams 的 margin 值来移动 View。



## View 的事件分发机制

### 1、点击事件的传递规则

点击事件的分发过程由三个方法共同完成：`dispatchTouchEvent`、`onInterceptTouchEvent`、`onTouchEvent`。

* 同一个事件序列指从手指接触屏幕开始到手指离开屏幕结束。即有 ACTION_DOWN 事件开始，中间包含数量不定的 ACTION_MOVE，以 ACTION_UP 结束。
* 通常一个时间序列只能被一个 View 拦截并消耗。
* 如果某个 View 决定拦截，那么这个时间序列都只能由它来处理，并且它的 `onInterceptTouchEvent` 方法不会再被调用。
* 如果某个 View 不消耗 ACTION_DOWN 事件，那么这个事件序列的后续事件都不会再交给它处理，并且事件将重新交由父元素处理，即父元素的 `onTouchEvent` 方法会被调用。
* 如果 View 不消耗除 ACTION_DOWN 之外的其他事件，那么这个事件序列将消失，父元素的 `onTouchEvent` 方法不会被调用，当前 View 将持续收到后续事件，最终这个点击事件将由 Activity 处理。
* ViewGroup 默认不拦截任何事件。
* View 没有 `onInterceptTouchEvent` 方法，一旦某个点击事件传递给它，它的 `onTouchEvent` 方法将被调用。
* View 的 `onTouchEvent` 方法默认会消耗事件，除非它是不可点击的（clickable 和 longClickable 同时为 false）。
* View 的 `enable` 属性不影响 `onTouchEvent` 的默认返回值。
* `onClick` 发生的前提是当前 View 可点击，并且它收到了 ACTION_DOWN 和 ACTION_UP 事件。
* 事件传递过程总是先传递给父元素，然后再由父元素分发给子元素。

### 2、源码分析

#### Activity 对点击事件的分发过程

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

TODO