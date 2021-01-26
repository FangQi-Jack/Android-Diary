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

#### measure

