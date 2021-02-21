## Handler

通过 handler 发送消息，通常可以调用它的 `sendMessage` 方法，通过查看源码可以发现，最终都是调用了 `sendMessageAtTime` 方法：

```java
/**
 * Enqueue a message into the message queue after all pending messages
 * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
 * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
 * Time spent in deep sleep will add an additional delay to execution.
 * You will receive it in {@link #handleMessage}, in the thread attached
 * to this handler.
 * 
 * @param uptimeMillis The absolute time at which the message should be
 *         delivered, using the
 *         {@link android.os.SystemClock#uptimeMillis} time-base.
 *         
 * @return Returns true if the message was successfully placed in to the 
 *         message queue.  Returns false on failure, usually because the
 *         looper processing the message queue is exiting.  Note that a
 *         result of true does not mean the message will be processed -- if
 *         the looper is quit before the delivery time of the message
 *         occurs then the message will be dropped.
 */
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
      msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}

/**
 * Handle system messages here.
 */
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
      handleCallback(msg);
    } else {
      if (mCallback != null) {
        if (mCallback.handleMessage(msg)) {
          return;
        }
      }
      handleMessage(msg);
    }
}

/**
 * Callback interface you can use when instantiating a Handler to avoid
 * having to implement your own subclass of Handler.
 */
public interface Callback {
    /**
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    boolean handleMessage(@NonNull Message msg);
}
```

该方法中调用了 `enqueueMessage` 方法，最终调用了 `MessageQueue` 的 `enqueueMessage` 方法。在 `dispatchMessage` 方法中，首先判断 `Message` 的 `callback` 是否为空，不为空就会调用 `handleCallback` 方法处理，这里的 `callback` 就是通过调用 `Handler.post()` 方法传递的 `Runnable` 对象。然后判断 `mCallback` 是否为空，如果不为空则调用它的 `handleMessage` 方法处理，这个 `mCallback` 是在初始化 `Handler` 时传入的，如果不想通过派生子类的方式实现创建 `Handler` 可以通过 `Callback` 来实现。最后调用了 `handleMessage` 方法。



## MessageQueue

```java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    synchronized (this) {
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}

Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
      return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
      if (nextPollTimeoutMillis != 0) {
        Binder.flushPendingCommands();
      }

      nativePollOnce(ptr, nextPollTimeoutMillis);

      synchronized (this) {
        // Try to retrieve the next message.  Return if found.
        final long now = SystemClock.uptimeMillis();
        Message prevMsg = null;
        Message msg = mMessages;
        if (msg != null && msg.target == null) {
          // Stalled by a barrier.  Find the next asynchronous message in the queue.
          do {
            prevMsg = msg;
            msg = msg.next;
          } while (msg != null && !msg.isAsynchronous());
        }
        if (msg != null) {
          if (now < msg.when) {
            // Next message is not ready.  Set a timeout to wake up when it is ready.
            nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
          } else {
            // Got a message.
            mBlocked = false;
            if (prevMsg != null) {
              prevMsg.next = msg.next;
            } else {
              mMessages = msg.next;
            }
            msg.next = null;
            if (DEBUG) Log.v(TAG, "Returning message: " + msg);
            msg.markInUse();
            return msg;
          }
        } else {
          // No more messages.
          nextPollTimeoutMillis = -1;
        }

        // Process the quit message now that all pending messages have been handled.
        if (mQuitting) {
          dispose();
          return null;
        }

        // If first time idle, then get the number of idlers to run.
        // Idle handles only run if the queue is empty or if the first message
        // in the queue (possibly a barrier) is due to be handled in the future.
        if (pendingIdleHandlerCount < 0
            && (mMessages == null || now < mMessages.when)) {
          pendingIdleHandlerCount = mIdleHandlers.size();
        }
        if (pendingIdleHandlerCount <= 0) {
          // No idle handlers to run.  Loop and wait some more.
          mBlocked = true;
          continue;
        }

        if (mPendingIdleHandlers == null) {
          mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
        }
        mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
      }

      // Run the idle handlers.
      // We only ever reach this code block during the first iteration.
      for (int i = 0; i < pendingIdleHandlerCount; i++) {
        final IdleHandler idler = mPendingIdleHandlers[i];
        mPendingIdleHandlers[i] = null; // release the reference to the handler

        boolean keep = false;
        try {
          keep = idler.queueIdle();
        } catch (Throwable t) {
          Log.wtf(TAG, "IdleHandler threw exception", t);
        }

        if (!keep) {
          synchronized (this) {
            mIdleHandlers.remove(idler);
          }
        }
      }

      // Reset the idle handler count to 0 so we do not run them again.
      pendingIdleHandlerCount = 0;

      // While calling an idle handler, a new message could have been delivered
      // so go back and look again for a pending message without waiting.
      nextPollTimeoutMillis = 0;
    }
}
```

`MessageQueue` 主要包含两个操作：`enqueueMessage` 和 `next`，分别对应插入消息和取出消息。`MessageQueue` 内部通过单链表存储消息，因为单链表在插入和删除操作上效率较高。其中 `enqueueMessage` 方法主要是向单链表中插入一条消息，而 `next` 方法用于从单链表中取出一条消息处理，并将其中链表中移除。



## Looper

通过调用 `Looper` 的 `loop` 方法启动消息循环机制。在非主线程中使用 `Handler` 需要先创建 `Looper`，通过调用 `Looper.prepare()` 可以创建 `Looper`，然后调用 `loop` 方法启动消息循环。

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    if (me.mInLoop) {
        Slog.w(TAG, "Loop again would have the queued messages be executed"
                + " before this one completed.");
    }

    me.mInLoop = true;
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    // Allow overriding a threshold with a system prop. e.g.
    // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
    final int thresholdOverride =
            SystemProperties.getInt("log.looper."
                    + Process.myUid() + "."
                    + Thread.currentThread().getName()
                    + ".slow", 0);

    boolean slowDeliveryDetected = false;

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        // Make sure the observer won't change while processing a transaction.
        final Observer observer = sObserver;

        final long traceTag = me.mTraceTag;
        long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
        long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
        if (thresholdOverride > 0) {
            slowDispatchThresholdMs = thresholdOverride;
            slowDeliveryThresholdMs = thresholdOverride;
        }
        final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
        final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

        final boolean needStartTime = logSlowDelivery || logSlowDispatch;
        final boolean needEndTime = logSlowDispatch;

        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }

        final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
        final long dispatchEnd;
        Object token = null;
        if (observer != null) {
            token = observer.messageDispatchStarting();
        }
        long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
        try {
          	// 调用 Handler 的 dispatchMessage 方法处理消息
            msg.target.dispatchMessage(msg);
            if (observer != null) {
                observer.messageDispatched(token, msg);
            }
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } catch (Exception exception) {
            if (observer != null) {
                observer.dispatchingThrewException(token, msg, exception);
            }
            throw exception;
        } finally {
            ThreadLocalWorkSource.restore(origWorkSource);
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        if (logSlowDelivery) {
            if (slowDeliveryDetected) {
                if ((dispatchStart - msg.when) <= 10) {
                    Slog.w(TAG, "Drained");
                    slowDeliveryDetected = false;
                }
            } else {
                if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                        msg)) {
                    // Once we write a slow delivery log, suppress until the queue drains.
                    slowDeliveryDetected = true;
                }
            }
        }
        if (logSlowDispatch) {
            showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
        }

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        msg.recycleUnchecked();
    }
}
```

在 `loop` 方法中有一个死循环，它不断调用 `MessageQueue` 的 `next` 方法取出消息，当消息为空时结束循环。随后执行 `msg.target.dispatchMessage(msg)` ，其中 `msg.target` 就是 `Handler`，此时消息传递到 `Handler` 的 `dispatchMessage` 方法处理。