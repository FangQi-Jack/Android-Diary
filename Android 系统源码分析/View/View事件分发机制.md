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
* 通常一个事件序列只能被一个 View 拦截并消耗。
* 如果某个 View 决定拦截，那么这个事件序列都只能由它来处理，并且它的 `onInterceptTouchEvent` 方法不会再被调用。
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

Activity 首先会将事件交由 Window 来进行分发，即 PhoneWindow 的 `superDispatchTouchEvent` 方法：

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

PhoneWindow 直接将事件交给了 DecorView 处理。因此，事件已经传递到了 Activity 的根 View 这里。

#### 根 View 对点击事件的分发过程

根 View 一般都是 ViewGroup，那么看 ViewGroup 的 `dispatchTouchEvent` 方法：

```java
// First touch target in the linked list of touch targets.
@UnsupportedAppUsage
private TouchTarget mFirstTouchTarget;


@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
  ...
    // Handle an initial down.
    if (actionMasked == MotionEvent.ACTION_DOWN) {
      // Throw away all previous state when starting a new touch gesture.
      // The framework may have dropped the up or cancel event for the previous gesture
      // due to an app switch, ANR, or some other state change.
      cancelAndClearTouchTargets(ev);
      resetTouchState();
    }
  
  	// Check for interception.
    final boolean intercepted;
    if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
      final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
      if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
      } else {
        intercepted = false;
      }
    } else {
      // There are no touch targets and this action is not an initial down
      // so this view group continues to intercept touches.
      intercepted = true;
    }
  
  ...
  
  		final View[] children = mChildren;
      for (int i = childrenCount - 1; i >= 0; i--) {
        final int childIndex = getAndVerifyPreorderedIndex(
          childrenCount, i, customOrder);
        final View child = getAndVerifyPreorderedView(
          preorderedList, children, childIndex);
        if (!child.canReceivePointerEvents()
            || !isTransformedTouchPointInView(x, y, child, null)) {
          continue;
        }

        newTouchTarget = getTouchTarget(child);
        if (newTouchTarget != null) {
          // Child is already receiving touch within its bounds.
          // Give it the new pointer in addition to the ones it is handling.
          newTouchTarget.pointerIdBits |= idBitsToAssign;
          break;
        }

        resetCancelNextUpFlag(child);
        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
          // Child wants to receive touch within its bounds.
          mLastTouchDownTime = ev.getDownTime();
          if (preorderedList != null) {
            // childIndex points into presorted list, find original index
            for (int j = 0; j < childrenCount; j++) {
              if (children[childIndex] == mChildren[j]) {
                mLastTouchDownIndex = j;
                break;
              }
            }
          } else {
            mLastTouchDownIndex = childIndex;
          }
          mLastTouchDownX = ev.getX();
          mLastTouchDownY = ev.getY();
          newTouchTarget = addTouchTarget(child, idBitsToAssign);
          alreadyDispatchedToNewTouchTarget = true;
          break;
        }

        // The accessibility focus didn't handle the event, so clear
        // the flag and do a normal dispatch to all children.
        ev.setTargetAccessibilityFocus(false);
      }
  
  ...
  
  	// Dispatch to touch targets.
    if (mFirstTouchTarget == null) {
      // No touch targets so treat this as an ordinary view.
      handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                              TouchTarget.ALL_POINTER_IDS);
    }
  ...
}

/**
 * Resets all touch state in preparation for a new cycle.
 */
private void resetTouchState() {
  clearTouchTargets();
  resetCancelNextUpFlag(this);
  mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
  mNestedScrollAxes = SCROLL_AXIS_NONE;
}

/**
 * Adds a touch target for specified child to the beginning of the list.
 * Assumes the target child is not already present.
 */
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
  final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
  target.next = mFirstTouchTarget;
  mFirstTouchTarget = target;
  return target;
}

/**
 * Transforms a motion event into the coordinate space of a particular child view,
 * filters out irrelevant pointer ids, and overrides its action if necessary.
 * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
 */
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
  final boolean handled;
  
  	if (child == null) {
      handled = super.dispatchTouchEvent(event);
    } else {
      handled = child.dispatchTouchEvent(event);
    }
  
  return handled;
}
```

`mFirstTouchTarget` 是 ViewGroup 中成功处理事件的子元素，当子 View 处理了事件时，`mFirstTouchTarget` 会被指向这个子 View。从源码中可以看到，如果事件为 ACTION_DOWN 或者 `mFirstTouchTarget` 为 null 时，会调用 `onInterceptTouchEvent` 询问 ViewGroup 是否要拦截事件，一旦 ViewGroup 拦截了事件，事件将不会传递到子 View，`mFirstTouchTarget != null` 就不成立，因此后续的 ACTION_MOVE 和 ACTION_UP 事件都不会调用 `onInterceptTouchEvent` 方法。 如果子 View 调用了 `requestDisallowInterceptTouchEvent` 方法，ViewGroup 将无法拦截除了 ACTION_DOWN 之外的所有事件，因为如果事件为 ACTION_DOWN 事件，ViewGroup 会执行 `resetTouchState` 方法清空 FLAG_DISALLOW_INTERCEPT 标记位。紧接着，如果 ViewGroup 没有拦截事件，那么事件将分发给子 View 处理，遍历所有子 View，判断子 View 是否能接收事件（子 View 当前是否在播放动画、点击事件的坐标是否落在子 View 区域内），然后调用 `dispatchTransformedTouchEvent` 方法，该方法内部实际调用了子 View 的 `dispatchTouchEvent` 方法，这样事件就传递到了子 View，当子 View 的 `dispatchTouchEvent` 方法返回 `true` 时，那么 `addTouchTarget` 方法将被执行，在这个方法中，`mFirstTouchTarget` 将被赋值，指向处理事件的 view。如果遍历了所有子 View 后事件仍然没有被消费（1、ViewGroup 没有子 View；2、子 View 的 `dispatchTouchEvent` 返回了 `false`），此时，事件将由 ViewGroup 处理。

#### View 对点击事件的处理过程

```java
public boolean dispatchTouchEvent(MotionEvent event) {
  ...
  if (onFilterTouchEventForSecurity(event)) {
    if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
      result = true;
    }
    //noinspection SimplifiableIfStatement
    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnTouchListener != null
        && (mViewFlags & ENABLED_MASK) == ENABLED
        && li.mOnTouchListener.onTouch(this, event)) {
      result = true;
    }

    if (!result && onTouchEvent(event)) {
      result = true;
    }
  }
  ...
  return result;
}

public boolean onTouchEvent(MotionEvent event) {
  ...
  final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

  if ((viewFlags & ENABLED_MASK) == DISABLED) {
    if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
      setPressed(false);
    }
    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
    // A disabled view that is clickable still consumes the touch
    // events, it just doesn't respond to them.
    return clickable;
  }
  if (mTouchDelegate != null) {
    if (mTouchDelegate.onTouchEvent(event)) {
      return true;
    }
  }
  if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
    switch (action) {
        case MotionEvent.ACTION_UP:
        	mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        if ((viewFlags & TOOLTIP) == TOOLTIP) {
          handleTooltipUp();
        }
        if (!clickable) {
          removeTapCallback();
          removeLongPressCallback();
          mInContextButtonPress = false;
          mHasPerformedLongPress = false;
          mIgnoreNextUpEvent = false;
          break;
        }
        boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
        if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
          // take focus if we don't have it already and we should in
          // touch mode.
          boolean focusTaken = false;
          if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
            focusTaken = requestFocus();
          }

          if (prepressed) {
            // The button is being released before we actually
            // showed it as pressed.  Make it show the pressed
            // state now (before scheduling the click) to ensure
            // the user sees it.
            setPressed(true, x, y);
          }

          if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
            // This is a tap, so remove the longpress check
            removeLongPressCallback();

            // Only perform take click actions if we were in the pressed state
            if (!focusTaken) {
              // Use a Runnable and post this rather than calling
              // performClick directly. This lets other visual state
              // of the view update before click actions start.
              if (mPerformClick == null) {
                mPerformClick = new PerformClick();
              }
              if (!post(mPerformClick)) {
                performClickInternal();
              }
            }
          }

          if (mUnsetPressedState == null) {
            mUnsetPressedState = new UnsetPressedState();
          }

          if (prepressed) {
            postDelayed(mUnsetPressedState,
                        ViewConfiguration.getPressedStateDuration());
          } else if (!post(mUnsetPressedState)) {
            // If the post failed, unpress right now
            mUnsetPressedState.run();
          }

          removeTapCallback();
        }
        mIgnoreNextUpEvent = false;
        break;
        ...
    }
  }
  ...
}
```

在 View（不包括 ViewGroup）的 `dispatchTouchEvent` 方法中，首先会判断是否设置了 `OnTouchListener` ，如果设置了并且 `onTouch` 方法返回了 `true`，那么将不再执行 `onTouchEvent` 方法。从 View 的 `onTouchEvent` 方法可以看出，即使 View 处于 `DISABLED` 状态，它依然会消耗事件。此外，如果设置了 `TouchDelegate`，也会执行它的 `onTouchEvent` 方法。接着，当事件为 ACTION_UP 时，只要 `CLICKABLE` 和 `LONG_CLICKABLE` 中有一个为 `true`，View 都会消耗这个事件，还会执行 `performClick` 方法，如果设置了 `OnClickListener`，那么将在 `performClick` 方法中调用 `onClick` 方法。

> 优先级：`OnTouchListener` > `onTouchEvent` > `OnClickListener`

