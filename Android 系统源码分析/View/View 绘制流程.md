# **View 绘制流程**

View 的绘制包括：measure、layout、draw。(以下源码来自 API 30)

### ViewRoot

ViewRoot 对应 ViewRootImpl，它是连接 WindowManager 和 DecorView 的纽带，View 的绘制流程均有 ViewRoot 完成。在 WindowManagerGlobal 的 addView 方法中会创建 ViewRootImpl。View 的绘制是从 performTraversals 开始的，该方法会依次执行 performMeasure 、 performLayout 、 performDraw 三个方法从而开始 View 的测量、布局和绘制。`performMeasure` 中会调用 `measure` ，`measure` 中会调用 `onMeasure`，在 `onMeasure` 中会调用子元素的 `measure`，由此进行子元素的绘制流程。`performLayout` 和 `performDraw` 也是类似的。



### MeasureSpec

MeasureSpec 在很大程度上决定了 View 的尺寸，父 View 会影响子 View 的 MeasureSpec，测量过程中，View 的 LayoutParams 会根据父 View 施加的规则转换成对应的 MeasureSpec，然后根据 MeasureSpec 测量出 View 的宽高。MeasureSpec 代表一个 32 位的 int 值，高 2 位代表 SpecMode（测量模式），低 30 位代表 SpecSize。

```java
public static class MeasureSpec {

  public static final int UNSPECIFIED = 0 << MODE_SHIFT;

  public static final int EXACTLY     = 1 << MODE_SHIFT;

  public static final int AT_MOST     = 2 << MODE_SHIFT;

  public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                    @MeasureSpecMode int mode) {
    if (sUseBrokenMakeMeasureSpec) {
      return size + mode;
    } else {
      return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
  }
  
  @MeasureSpecMode
  public static int getMode(int measureSpec) {
    //noinspection ResourceType
    return (measureSpec & MODE_MASK);
  }
  
  public static int getSize(int measureSpec) {
    return (measureSpec & ~MODE_MASK);
  }
  
  ...
}
```

##### UNSPECIFIED

父容器不会对子 View 做任何限制，子 View 可以想多大就多大。

##### 2、EXACTLY

父容器会指定一个精确值给子 View，这时候 View 的最终大小就是 SpecSize 所指定的值。对应 LayoutParams 的 match_parent 和指定数值。

##### 3、AT_MOST

子 View 不能超过父容器指定的大小。对应 LayoutParams 的 wrap_content。

#### MeasureSpec 和 LayoutParams

对应 DecorView，它的 MeasureSpec 由窗口尺寸和其自身的 LayoutParams 共同决定，而对于普通 View，它的 MeasureSpec 由父容器的 MeasureSpec 和自身的 LayoutParams 共同决定。

###### 1、DecorView 的 MeasureSpec 产生过程

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
  ...
    
  private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    ...
    if (!goodMeasure) {
      childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
      childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
      if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
        windowSizeMayChange = true;
      }
    }
    ...
  }
          
  ...
    
  /**
   * Figures out the measure spec for the root view in a window based on it's
   * layout params.
   *
   * @param windowSize
   *            The available width or height of the window
   *
   * @param rootDimension
   *            The layout params for one dimension (width or height) of the
   *            window.
   *
   * @return The measure spec to use to measure the root view.
   */
  private static int getRootMeasureSpec(int windowSize, int rootDimension) {
      int measureSpec;
      switch (rootDimension) {

      case ViewGroup.LayoutParams.MATCH_PARENT:
          // Window can't resize. Force root view to be windowSize.
          measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
          break;
      case ViewGroup.LayoutParams.WRAP_CONTENT:
          // Window can resize. Set max size for root view.
          measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
          break;
      default:
          // Window wants to be an exact size. Force root view to be that size.
          measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
          break;
      }
      return measureSpec;
  }
  ...
}
```

###### 2、普通 View 的 MeasureSpec 的产生过程

ViewGroup 的 measureChildWithMargins 方法：

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

其中，在执行 child 的`measure` 方法前通过调用 `getChildMeasureSpec` 方法获取子 View 的 MeasureSpec，可以看到与父容器的 measureSpec、自身 LayoutParams 以及自身的 padding、margin 有关。

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```



### View 的绘制流程

#### measure 过程

##### 1、View 的 measure 过程

View 的 measure 方法完成了 View 的测量，在 measure 方法中会调用 onMeasure 方法：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
  boolean optical = isLayoutModeOptical(this);
  if (optical != isLayoutModeOptical(mParent)) {
    Insets insets = getOpticalInsets();
    int opticalWidth  = insets.left + insets.right;
    int opticalHeight = insets.top  + insets.bottom;

    measuredWidth  += optical ? opticalWidth  : -opticalWidth;
    measuredHeight += optical ? opticalHeight : -opticalHeight;
  }
  setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
  mMeasuredWidth = measuredWidth;
  mMeasuredHeight = measuredHeight;

  mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}

public static int getDefaultSize(int size, int measureSpec) {
  int result = size;
  int specMode = MeasureSpec.getMode(measureSpec);
  int specSize = MeasureSpec.getSize(measureSpec);

  switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
      result = size;
      break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
      result = specSize;
      break;
  }
  return result;
}

protected int getSuggestedMinimumHeight() {
  return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
}

protected int getSuggestedMinimumWidth() {
  return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

在 onMeasure 方法中调用 `setMeasuredDimension` 方法来设置 View 宽高的测量值。参数中调用的方法 `getDefaultSize` 方法中通过判断 `specMode` 返回对应的值，如果为 `AT_MOST` 或者 `EXACTLY` 则返回 `specSize` ，如果是 `UNSPECIFIED` 则返回参数中传入的 size，也即 `getSuggestedMinimumWidth()` 、 `getSuggestedMinimumHeight()` 的值。而在这两个方法中，如果 View 没有设置 background 则返回它的最小宽度或最小高度，对应的是  `android:minWidth` 和 `android:minHeight` 设置的值，如果没有设置则为 0，如果设置了 background 则会返回 最小宽高与 background 的 `getMinimumHeight()`/`getMinimumWidth()`中的较大者，而 `Drawable` 的两个方法如下：

```java
public int getMinimumWidth() {
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}

public int getMinimumHeight() {
  final int intrinsicHeight = getIntrinsicHeight();
  return intrinsicHeight > 0 ? intrinsicHeight : 0;
}
```

返回就是 `Drawable` 的固有宽高，`ShapeDrawable` 没有固有宽高，`BitmapDrawable` 的固有宽高为图片的尺寸。因为，对我们来说，View 的宽高由 `specSize` 来决定。**因此在直接继承 View 自定义控件的时候需要对 `wrap_content` 做特殊处理。因为如果使用 `wrap_content`，它的 `specMode` 是 `AT_MOST`，这种情况下它的测量宽高等于 `specSize`，此时的 `specSize` 也就等于父容器可使用空间大小，如果不做处理，那么相当于使用 `match_parent`。**

##### 2、ViewGroup 的 measure 过程

ViewGroup 不仅要完成自身的 measure 过程，还需要遍历执行所有子元素的 measure 方法。ViewGroup 中并没有重新 View 的 `onMeasure` 方法，而是提供了 `measureChildren` 方法：

```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}

protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
  final LayoutParams lp = child.getLayoutParams();

  final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                                                        mPaddingLeft + mPaddingRight, lp.width);
  final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                                                         mPaddingTop + mPaddingBottom, lp.height);

  child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

主要过程就是遍历所有子 View，取出子 View 的 `LayoutParams` ，并执行 `getChildMeasureSpec` 构建子 View 的 `MeasureSpec`，然后执行子 View 的 `measure` 方法。ViewGroup 并没有定义 `onMeasure` 方法的实现，因为 ViewGroup 并不像 View 那样只有单一元素，ViewGroup 中的元素可能不尽相同，因此 ViewGroup 是一个抽象类，`onMeasure` 需要具体子类具体实现。以最常用的 LinearLayout 为例：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```

这里根据 `LinearLayout` 的方向不同执行对应的方法。主要过程：遍历子元素，并对其执行  `measureChildBeforeLayout` 方法，在此方法内部会执行子元素的 `measure` 方法，开始子元素的 measure 过程；当 LinearLayout 的所有资源都策略完毕后，会根据子元素的情况来测量自身大小。对于竖直方向的 LinearLayout 而已，当它是 `match_parent` 或者具体数值时，测量过程和 View 一样，如果是 `wrap_content` 时，它的高度是所有子 View 的高度之和加上竖直方向上的 padding，但仍然不能超过父容器的剩余空间大小。



#### layout 过程

layout 过程确定 View 的位置。

```java
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);

        if (shouldDrawRoundScrollbar()) {
            if(mRoundScrollbarRenderer == null) {
                mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
            }
        } else {
            mRoundScrollbarRenderer = null;
        }

        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    final boolean wasLayoutValid = isLayoutValid();

    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

    if (!wasLayoutValid && isFocused()) {
        mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
        if (canTakeFocus()) {
            // We have a robust focus, so parents should no longer be wanting focus.
            clearParentsWantFocus();
        } else if (getViewRootImpl() == null || !getViewRootImpl().isInLayout()) {
            // This is a weird case. Most-likely the user, rather than ViewRootImpl, called
            // layout. In this case, there's no guarantee that parent layouts will be evaluated
            // and thus the safest action is to clear focus here.
            clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            clearParentsWantFocus();
        } else if (!hasParentWantsFocus()) {
            // original requestFocus was likely on this view directly, so just clear focus
            clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
        }
        // otherwise, we let parents handle re-assigning focus during their layout passes.
    } else if ((mPrivateFlags & PFLAG_WANTS_FOCUS) != 0) {
        mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
        View focused = findFocus();
        if (focused != null) {
            // Try to restore focus as close as possible to our starting focus.
            if (!restoreDefaultFocus() && !hasParentWantsFocus()) {
                // Give up and clear focus once we've reached the top-most parent which wants
                // focus.
                focused.clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            }
        }
    }

    if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
        mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
        notifyEnterOrExitForAutoFillIfNeeded(true);
    }

    notifyAppearedOrDisappearedForContentCaptureIfNeeded(true);
}
```

在 `layout` 方法中，首先执行 `setFrame` 方法确定 View 的四个顶点的位置，即 `mLeft`、`mRight`、`mTop` 和 `mBottom` 的值，然后会调用 `onLayout` 方法，该方法由具体布局自己实现，View 和 ViewGroup 中都没有实现该方法。同样以 `LinearLayout` 为例：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}

void layoutVertical(int left, int top, int right, int bottom) {
  final int paddingLeft = mPaddingLeft;

  int childTop;
  int childLeft;

  // Where right end of child should go
  final int width = right - left;
  int childRight = width - mPaddingRight;

  // Space available for child
  int childSpace = width - paddingLeft - mPaddingRight;

  final int count = getVirtualChildCount();

  final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
  final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

  switch (majorGravity) {
    case Gravity.BOTTOM:
      // mTotalLength contains the padding already
      childTop = mPaddingTop + bottom - top - mTotalLength;
      break;

      // mTotalLength contains the padding already
    case Gravity.CENTER_VERTICAL:
      childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
      break;

    case Gravity.TOP:
    default:
      childTop = mPaddingTop;
      break;
  }

  for (int i = 0; i < count; i++) {
    final View child = getVirtualChildAt(i);
    if (child == null) {
      childTop += measureNullChild(i);
    } else if (child.getVisibility() != GONE) {
      final int childWidth = child.getMeasuredWidth();
      final int childHeight = child.getMeasuredHeight();

      final LinearLayout.LayoutParams lp =
        (LinearLayout.LayoutParams) child.getLayoutParams();

      int gravity = lp.gravity;
      if (gravity < 0) {
        gravity = minorGravity;
      }
      final int layoutDirection = getLayoutDirection();
      final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
      switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
        case Gravity.CENTER_HORIZONTAL:
          childLeft = paddingLeft + ((childSpace - childWidth) / 2)
            + lp.leftMargin - lp.rightMargin;
          break;

        case Gravity.RIGHT:
          childLeft = childRight - childWidth - lp.rightMargin;
          break;

        case Gravity.LEFT:
        default:
          childLeft = paddingLeft + lp.leftMargin;
          break;
      }

      if (hasDividerBeforeChildAt(i)) {
        childTop += mDividerHeight;
      }

      childTop += lp.topMargin;
      setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
      childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

      i += getChildrenSkipCount(child, i);
    }
  }
}

private void setChildFrame(View child, int left, int top, int width, int height) {
  child.layout(left, top, left + width, top + height);
}
```

同样根据不同方法执行不同的方法。同样的遍历所有子 View，并调用 `setChildFrame` 方法，该方法中直接调用了子 View 的 `layout` 方法， 确定子 View 的位置，并且 `childTop` 会不断增长，因此，后续的 View 会放在靠下的位置。

###### View 的测量宽高与实际宽高的关系？

```java
public final int getWidth() {
    return mRight - mLeft;
}

public final int getHeight() {
  return mBottom - mTop;
}
```

可以看到，`getWidth()` 和 `getHeight()` 返回的就是 View 的测量宽高。在默认实现中，View 的实际宽高与测量宽高相等，只是赋值时机不同，也存在特别的情况，两者并不相等。



### draw 过程

draw 过程作用是绘制，主要有以下过程：

* 绘制 background
* 绘制内容（执行 `onDraw`）
* 绘制子元素（执行 `dispatchDraw`，该方法会遍历所有子元素并执行它的 `draw` 方法）
* 绘制装饰，如：foreground、scrollbars
* 绘制默认高亮焦点

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     *      7. If necessary, draw the default focus highlight
     */

    // Step 1, draw the background, if needed
    int saveCount;

    drawBackground(canvas);

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        drawAutofilledHighlight(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);

        if (isShowingLayoutBounds()) {
            debugDrawFocus(canvas);
        }

        // we're done...
        return;
    }
  ...
}

/**
     * If this view doesn't do any drawing on its own, set this flag to
     * allow further optimizations. By default, this flag is not set on
     * View, but could be set on some View subclasses such as ViewGroup.
     *
     * Typically, if you override {@link #onDraw(android.graphics.Canvas)}
     * you should clear this flag.
     *
     * @param willNotDraw whether or not this View draw on its own
     */
public void setWillNotDraw(boolean willNotDraw) {
  setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```

当 View 自身不需要绘制任何内容是，通过 `setWillNotDraw` 将标记位设置为 true，可以让系统进行优化，ViewGroup 默认为 true。如果确定了需要绘制内容，应该显示的清除该标记位。