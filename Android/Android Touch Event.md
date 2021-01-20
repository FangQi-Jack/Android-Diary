## Android Touch Event

* In <font color="red">ViewGroup</font>([https://developer.android.com/training/gestures/viewgroup.html#intercept](https://developer.android.com/training/gestures/viewgroup.html#intercept "原网址"))

* [http://codetheory.in/understanding-android-input-touch-events/](http://codetheory.in/understanding-android-input-touch-events/ "可参考文档")

> The <font color="blue">onInterceptTouchEvent()</font> method is called whenever a touch event is detected on the surface of a <font color="blue">*ViewGroup*</font>, including on the surface of its children. If <font color="blue">onInterceptTouchEvent()</font> returns *true*, the <font color="blue">*MotionEvent*</font> is intercepted, meaning it will be not be passed on to the child, but rather to the <font color="blue">*onTouchEvent()*</font> method of the parent.

> The <font color="blue">onInterceptTouchEvent()</font> method gives a parent the chance to see any touch event before its children do. If you return **true** from <font color="blue">onInterceptTouchEvent()</font>, the child view that was previously handling touch events receives an <font color="blue">ACTION_CANCEL</font>, and the events from that point forward are sent to the parent's <font color="blue">onTouchEvent()</font> method for the usual handling. <font color="blue">onInterceptTouchEvent()</font> can also return **false** and simply spy on events as they travel down the view hierarchy to their usual targets, which will handle the events with their own <font color="blue">onTouchEvent()</font>.

> *Note that* ViewGroup also provides a <font color="blue">requestDisallowInterceptTouchEvent()</font> method. The <font color="blue">ViewGroup</font> calls this method when a child does not want the parent and its ancestors to intercept touch events with <font color="blue">onInterceptTouchEvent()</font>.

> * **important methods:** (Use ViewConfiguration Constants) 
	 * <font color="blue">getScaledTouchSlop()</font>: "Touch slop" refers to the distance in pixels a user's touch can wander before the gesture is interpreted as scrolling. Touch slop is typically used to prevent accidental scrolling when the user is performing some other touch operation, such as touching on-screen elements.
	 * <font color="blue">getScaledMinimumFlingVelocity()</font>: return the minimum velocity to initiate a fling, as measured in pixels per second.
	 * <font color="blue">getScaledMaximumFlingVelocity()</font>: return the maximum velocity to initiate a fling, as measured in pixels per second.