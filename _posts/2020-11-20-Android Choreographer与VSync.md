---
layout:     post
title:      Choreographer与VSync机制
subtitle:
date:       2020-11-20
author:     parting_soul
header-img: img/vsync.jpg
catalog: true
tags:
    - Android
    - 源码
    - 笔记
---

> 源码基于Android 10.0

### 问题

当一个控件内容或者宽高发生变化，到显示到屏幕上经历一个怎样的过程

### 一. 基本概念

- 刷新率： 每秒钟屏幕刷新的次数
- 帧率：每秒钟图像绘制帧的次数
- 当帧率小于屏幕的刷新率就可能会发生卡顿现象(同一帧在多个刷新周期内被绘制)，帧率大于屏幕刷新率会发生屏幕撕裂的问题
- Android 中由于要处理输入、动画以及测量、绘制等，帧率达到60 FPS时才不会造成卡顿，也就是尽可能保证在16ms内完成一帧的绘制

###  二. 执行流程

一个View内容或者宽高发生变化，会调用View的 **invalid() **  或者 **requestLayout()** 方法，例如TextView，通过**setText()** 方法设置文本内容，内部最终调用了 **requestLayout()**以及 **invalid() ** 方法

TextView.java

```java
private void setText(CharSequence text, BufferType type,
                         boolean notifyBefore, int oldlen) {
     ....

    if (mLayout != null) {
        checkForRelayout();
    }

 }

private void checkForRelayout() {

	...
	requestLayout();
  invalidate();
}
```

可以看到首先调用了 **View的requestLayout()** 方法，然后在该方法内向上追溯，调用父容器的**requestLayout()**方法

 View.java

```java
public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();

    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null && viewRoot.isInLayout()) {
            if (!viewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
        mAttachInfo.mViewRequestingLayout = this;
    }

    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    // 调用父容器的requestLayout
    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```

TextView的父容器ViewGroup没有重写 **requestLayout()** ，所以还是调用父容器父类View的方法，一直向上追溯，直到顶层容器DecorView，DecorView的mParent为ViewRootImp



Activity执行onResume后，会创建ViewRootImp对象，并且和DecorVIew进行关联

WindowManagerGlobal.java

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {

    ... 

    ViewRootImpl root;
    View panelParentView = null;
 	 
    ...
    // 创建 ViewRootImpl
    root = new ViewRootImpl(view.getContext(), display);
    // ViewRootImp与DecorView进行关联，其中将DecorView的parent设置为ViewRootImp
    root.setView(view, wparams, panelParentView);
}
```

在ViewRootImp的**setView()方法** 中将该ViewRootImp设置为DecorView的ViewParent

ViewRootImp.java

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	 synchronized (this) {
        if (mView == null) {
            mView = view;
        	...

        	// 请求布局
        	requestLayout();
        
         	// 添加windows至窗口
            res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                            mTempInsets);
        
            ...

            // 将当前ViewRootImp设置为DecorView的ViewParent 
            view.assignParent(this);
        }    	
     }
}
```

再次回到DecorView，之前TextView设置内容后调用**requestLayout() **向上追溯到DecorView，DecorView同样会调用ViewParent的**requestLayout()**方法(也就是ViewRootImp的requestLayout方法)

ViewRootImpl.java

```java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
    	// 检查是否是UI线程调用了该方法
        checkThread();
        mLayoutRequested = true;
        // 执行遍历
        scheduleTraversals();
    }
}
```

在ViewRootImp的**requestLayout() **中首先判断了调用线程，当不在UI线程更新UI时，就会抛出下述异常

```java
void checkThread() {
  if (mThread != Thread.currentThread()) {
    throw new CalledFromWrongThreadException(
      "Only the original thread that created a view hierarchy can touch its views.");
  }
}
```

但是一些时候在子线程更新UI并不会抛出异常，例如在 Activity onCreate中开启一个子线程去更新UI，并不会发生异常

![image-20201119234900362](http://img.partingsoul.cn//image-20201119234900362.png)

其实原因在上述流程中已经显示出来了，就是ViewRootImp这个对象是在Activity的onResume之后才被创建，虽然TextView更改了内容，调用**requestLayout() 方法**，但是此时DecorView的ViewParent为null，所以此时并不会抛出异常。



检查完线程状态后，调用**scheduleTraversals() 方法** 执行遍历。

- 用 mTraversalScheduled 这个标志位来防止短时间内重复进行遍历，也就是连续两次requestLayout，只有第一次有效；
- 提交异步任务之前，首先向MessageQueue提交一个同步屏障，作用是让异步消息首先执行
- 向编舞者提交一个异步任务

ViewRootImp.java

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;

        // 加入同步屏障，异步消息会首先执行，这里保证该遍历任务优先执行
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 向编舞者提交一个消息
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

首先看 mTraversalRunnable 这个任务，实现了Runable接口

```java
final class TraversalRunnable implements Runnable {
  @Override
  public void run() {
    doTraversal();
  }
}
```

run方法内部执行了 **doTraversal() 方法**

- 异步回调成功后，mTraversalScheduled 重新又置为false
- 移除同步屏障
- 调用 **performTraversals()方法**，开始执行测量，布局，绘制

这边流程虽然是通了，但是还是有一个疑问，Choreographer 这个类到底起了什么作用？

ViewRootImp.java

```java
void doTraversal() {
  if (mTraversalScheduled) {
    // 标志位重新置为false
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

来看下 Choreographer 这个类，大致介绍下一些主要的成员

这个类的作用主要是协调输入，动画以及绘制的时间。它定时接收一个同步脉冲，触发下一帧绘制的相关操作

Choreographer.java

```java
public final class Choreographer {

	// 一个线程对应一个 Choreographer对象
    private static final ThreadLocal<Choreographer> sThreadInstance =
        new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        Looper looper = Looper.myLooper();
        if (looper == null) {
            throw new IllegalStateException("The current thread must have a looper!");
        }
        Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
        if (looper == Looper.getMainLooper()) {
            mMainInstance = choreographer;
        }
        return choreographer;
	    }
	};


    // 处理四种类型的Callback消息
    private static final String[] CALLBACK_TRACE_TITLES = {
        "input", "animation", "insets_animation", "traversal", "commit"
	};


	// 存放Callback消息的队列数组，索引就是Callback 消息类型
	private final CallbackQueue[] mCallbackQueues;

	// 一个Handler 
	private final FrameHandler mHandler;

	
    private final class FrameHandler extends Handler {
		public FrameHandler(Looper looper) {
		    super(looper);
		}

		@Override
		public void handleMessage(Message msg) {
		    switch (msg.what) {
		        case MSG_DO_FRAME:
		        	// 执行下一帧的操作
		            doFrame(System.nanoTime(), 0);
		            break;
		        case MSG_DO_SCHEDULE_VSYNC:
		        	// 请求vsync信号
		            doScheduleVsync();
		            break;
		        case MSG_DO_SCHEDULE_CALLBACK:
		        	// 执行Callback任务(同样会请求vsync信号)
		            doScheduleCallback(msg.arg1);
		            break;
		    }
		}
	}


	// Callback 消息队列
	private final class CallbackQueue {
		// 单链表头
        private CallbackRecord mHead;

        public boolean hasDueCallbacksLocked(long now) {
            return mHead != null && mHead.dueTime <= now;
        }

        // 筛选出可以执行的消息列表
        public CallbackRecord extractDueCallbacksLocked(long now) {
            CallbackRecord callbacks = mHead;
            if (callbacks == null || callbacks.dueTime > now) {
                return null;
            }

            CallbackRecord last = callbacks;
            CallbackRecord next = last.next;
            while (next != null) {
                if (next.dueTime > now) {
                    last.next = null;
                    break;
                }
                last = next;
                next = next.next;
            }
            mHead = next;
            return callbacks;
        }


       	// 按执行时间排序，优先执行的放在表头
        @UnsupportedAppUsage
        public void addCallbackLocked(long dueTime, Object action, Object token) {
          ...
        }

        public void removeCallbacksLocked(Object action, Object token) {
            ...
        }
    }
}
```

还是之前那个例子，在ViewRootImp的 **doTraversal()方法**通过 Choreographer 提交了一个Runable任务，任务类型是 CALLBACK_TRAVERSAL

ViewRootImp.java

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;

     	...
        // 向编舞者提交一个消息
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}
```

ViewRootImp中通过post提交任务，最终调用了**postCallbackDelayedInternal() 方法**

- 首先计算任务的执行时间
- 将任务加入到对应的Callback队列
- 非延迟消息立刻执行
- 延迟消息通过Handler进行延迟处理

```java
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}

public void postCallbackDelayed(int callbackType,
        Runnable action, Object token, long delayMillis) {
    ... 
    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {

    synchronized (mLock) {
    	// 计算执行时间
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        // 将当前Callback类型的消息加入对应的队列
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
        	// 非延迟消息，立马执行
            scheduleFrameLocked(now);
        } else {
        	// 延迟执行，发送异步延迟消息
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

这边先看非延迟消息

- 这边同样做了防止重复调用的处理
- 通过是否使用vsync信号进行不同的处理

```java
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
        	// 使用vsync信号
            if (isRunningOnLooperThreadLocked()) {
            	// 在主线程中执行
                scheduleVsyncLocked();
            } else {
            	// 不在主线程中执行，通过handler发送至主线程
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
        	// 不使用vsync信号
            final long nextFrameTime = Math.max(
                    mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
           	// 发送执行下一帧的消息
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}
```

从Android 4.1开始，Google就引入了VSync机制，也就是使用VSync，这边UI的更新在主线程，就先看主线程的方式，这边直接请求了一个VSync信号

```java
private void scheduleVsyncLocked() {
  mDisplayEventReceiver.scheduleVsync();
}
```

若在其他线程中调用，则发送一个MSG_DO_SCHEDULE_VSYNC的异步消息，可以查看Handler的处理消息的回调，由代码可知，调用了 **doScheduleVsync()方法**

```java
private final class FrameHandler extends Handler {
  public FrameHandler(Looper looper) {
    super(looper);
  }

  @Override
  public void handleMessage(Message msg) {
    switch (msg.what) {
      case MSG_DO_FRAME:
        doFrame(System.nanoTime(), 0);
        break;
      case MSG_DO_SCHEDULE_VSYNC:
        doScheduleVsync();
        break;
      case MSG_DO_SCHEDULE_CALLBACK:
        doScheduleCallback(msg.arg1);
        break;
    }
  }
}
```

可以看到，若在子线程中发送Callback任务，则最后还是会转到主线程中去请求VSync信号量

```java
void doScheduleVsync() {
  synchronized (mLock) {
    if (mFrameScheduled) {
      scheduleVsyncLocked();
    }
  }
}
private void scheduleVsyncLocked() {
  mDisplayEventReceiver.scheduleVsync();
}
```

再来看看mDisplayEventReceiver对象对用的类 FrameDisplayEventReceiver

- VSync 信号到来时会调用该对象的 **onVsync() 方法**
- 由于当前对象实现了Runnable接口，收到VSync信号后，会发送一个异步消息，调用该对象run方法
- 在run方法中调用**doFrame()方法**，开始处理下一帧

FrameDisplayEventReceiver.java

```java
// VSync信号到来时会回调 onVsync方法
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }

      
        @Override
        public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
          	
            long now = System.nanoTime();
            if (timestampNanos > now) {
            	// 信号到来的时间大于当前时间，进行微调(存在疑问)
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                Log.w(TAG, "Already have a pending vsync event.  There should only be "
                        + "one at a time.");
            } else {
                mHavePendingVsync = true;
            }

            // 记录信号到来的时间
            mTimestampNanos = timestampNanos;
            mFrame = frame;

            // 发送一个异步消息，执行该对象的run方法
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
        	// 开始处理下一帧
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
}
```

到了doFrame方法

- 首先对开始执行的时间偏差进行调整，尽可能保证绘制和信号处于同步
- 保存当前执行帧的信息(信号到来时间，开始执行时间)
- 按顺序执行输入、动画、遍历、提交的回调

```java
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        if (!mFrameScheduled) {
            return; // no work to do
        }

        // 保存信号量到来的时间
        long intendedFrameTimeNanos = frameTimeNanos;
        // 当前的执行时间
        startNanos = System.nanoTime();
        // 得到执行时间与信号量到来时间的间隔
        final long jitterNanos = startNanos - frameTimeNanos;

        if (jitterNanos >= mFrameIntervalNanos) {
        	// 若时间间隔大于一个VSync周期，说明存在跳帧，计算跳了几个帧
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
            	// 跳过的帧数超过一个上限
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
            // 计算最后一帧执行完和Vsync到来时间的偏移量
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
         	// 由于需要将帧的绘制与VSync信号的周期保持同步，这边假定下一帧的开始执行时间等于最近一个Vsync信号到来的时间
            frameTimeNanos = startNanos - lastFrameOffset;
        }

        if (frameTimeNanos < mLastFrameTimeNanos) {
          	// 信号量到来时间比上一次时间小，这里是时间发生了错误
            scheduleVsyncLocked();
            return;
        }

        if (mFPSDivisor > 1) {
            long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
            if (timeSinceVsync < (mFrameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
                scheduleVsyncLocked();
                return;
            }
        }

        // 设置帧的信息(信号量到来的时间，真正开始执行的时间)
        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
        mFrameScheduled = false;
        mLastFrameTimeNanos = frameTimeNanos;
    }

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

        // 执行输入有关的Callback 
        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        // 执行动画有关的Callback
        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);

        // 执行遍历有关的Callback(测量，布局，绘制)
        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        // Commit有关Callback
        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

这边看遍历的回调执行

- 根据Callback类型得到消息队列，从消息队列中取出可以处理的Callback
- 可以处理的Callback是以单链表的方式组织的，遍历整个链表，执行run方法
- 回收所有被执行的CallbackRecord

```java
void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
       	// 根据Callback 类型得到对应的Callback队列，从Callback队列中取出可以执行的消息
        final long now = System.nanoTime();
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                now / TimeUtils.NANOS_PER_MS);
        if (callbacks == null) {
            return;
        }
        mCallbacksRunning = true;

      
    try {
    	// 遍历单链表，开始执行消息
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            // 开始执行
            c.run(frameTimeNanos);
        }
    } finally {
        synchronized (mLock) {
        	// 执行完，回收消息
            mCallbacksRunning = false;
            do {
                final CallbackRecord next = callbacks.next;
                recycleCallbackLocked(callbacks);
                callbacks = next;
            } while (callbacks != null);
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

再来看看CallbackRecord的run方法的逻辑,这边是遍历类型的Callback，所以是action为Runable类型，这个action就是通过postCallback方式提交的Runnable

```java
private static final class CallbackRecord {
  public CallbackRecord next;
  public long dueTime;
  public Object action; // Runnable or FrameCallback
  public Object token;

  @UnsupportedAppUsage
  public void run(long frameTimeNanos) {
    if (token == FRAME_CALLBACK_TOKEN) {
      ((FrameCallback)action).doFrame(frameTimeNanos);
    } else {
      ((Runnable)action).run();
    }
  }
}
```

执行TraversalRunnable的run方法

```java
final class TraversalRunnable implements Runnable {
  @Override
  public void run() {
    doTraversal();
  }
}
```

在doTraversal方法中调用了performTraversals，开始进行测量、布局、绘制

```java
void doTraversal() {
  if (mTraversalScheduled) {
    mTraversalScheduled = false;
    mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

    if (mProfile) {
      Debug.startMethodTracing("ViewAncestor");
    }
	
    // 执行遍历
    performTraversals();

    if (mProfile) {
      Debug.stopMethodTracing();
      mProfile = false;
    }
  }
}
```

至此，整个流程就清楚了

### 三. 总结

- 通过setText给TextView设置文本内容，最终会调用TextView的requestLayout
- TextView的requestLayout被调用后，会向上追溯，直到ViewRootImp
- 调用ViewRootImp的requestLayout，然后 调用 scheduleTraversals 开始处理遍历
- 通过 Runnable遍历任务提交到Choreographer，加入Callback 遍历队列
- 若当前任务需要立即执行，则直接请求VSync信号；否则，发送延迟消息，等到延迟消息被执行，同样请求VSync信号
- VSync 信号到达，FrameDisplayEventReceiver的onVsyn方法被回调
- 调整开始执行的时间偏差，依次执行 输入、动画、遍历等Callback任务
- 这边只考虑遍历任务，从队列中取出可以执行的遍历任务，然后执行
- ViewRootImp的mTraversalRunnable被执行
- 最终执行了ViewRootImp的 performTraversals，从而开始测量、布局、绘制



![vsync](http://img.partingsoul.cn//vsync.png)



参考

[Android图形显示系统（一)](https://www.jianshu.com/p/424918260fa9?open_source=weibo_search)

[“终于懂了” 系列：Android屏幕刷新机制—VSync、Choreographer 全面理解！](https://juejin.cn/post/6863756420380196877#heading-4)

