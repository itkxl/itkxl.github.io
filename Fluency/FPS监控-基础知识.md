## 一、引言
在日常App开发过程中，流畅的视觉体验是用户满意的基本保证。要实现这一点，需要对现实相关的内容，比如帧缓冲区、刷新率、帧率等有一个基本的认识。在本文中，我们将简单概述一下这些基本概念以及在Android平台上的实现，帮助后续相关监控能力的建设。

## 二、显示基础

### 2.1 刷新率 
在购买手机和显示器的时候，这是一个很重要的硬件参数。刷新率高意味着可以每秒钟更新更多次画面，之前的手机大多数为60Hz，即每秒钟刷新60次。随着技术发展，越来越多的设备开始采用更高的帧率，如90Hz,120Hz等，这样可以让屏幕显示的和动效更加平滑，减少视觉延迟&卡顿。
### 2.2 帧率
帧率，一般使用FPS(每秒帧数)，描述每秒钟能生成多少个页面，由CPU&GPU完成。

### 2.3 刷新率与帧率的区别
两者概念听起来较为相近，但是却有本质区别，两者更像是生产者和消费者的关系。
- 帧率：生产者，负责生成帧供屏幕进行展示，或者说供刷新率进行消费（可能说成生产者的速度更为恰当一些）。
- 刷新率：消费者，消费生成的帧的速度。

### 2.3 FrameBuffer
帧的生产和消费之间需要一个桥梁，其就是FrameBuffer。它本质上是一块内存区域，它用来临时存储整个屏幕即将显示的像素数据。

### 2.4 问题
前面介绍了刷新率、帧率、FrameBuffer的基本概念，刷新率和帧率是两个速度，既然是速度，可能出现值不匹配的情况。
- 刷新率高于帧率：当显示设备的刷新率（屏幕每秒更新的次数）高于帧率（设备每秒生成的画面数）时，屏幕有时候会不得不重复同事同一帧画面，直到下一帧准备好为止。会造成卡顿感。
- 帧率高于刷新率：当帧率高于刷新率，每秒生成的次数超过了屏幕每秒能更新的次数，这可能造成单次刷新期间FrameBuffer中同时有多个帧的信息，这将导致一个常见的问题，屏幕撕裂。

如何解决上述两者不同步的问题呢？接下来介绍垂直同步技术。

## 三、垂直同步（VSync）
### 3.1 什么是VSync?
VSync，全称Vertical Synachronization（垂直同步），是用来解决屏幕撕裂问题的一种技术。屏幕撕裂是指显示设备在刷新内容时，上一帧的内容与新一帧的内容不同步，造成的屏幕上部分区域显示上一帧的图片，而另外一部分显示的是新帧图像的现象。

### 3.2 VSync的工作原理
VSync的工作原理是将应用的图形渲染操作与显示器的刷新率进行同步。大多数的设备的显示器的刷新率是60Hz,即每秒刷新60次。开启VSync后，应用的渲染速度会被锁定，以确保每次渲染的结果都在显示器准备刷新的时候完成，减少撕裂现象。
换句话说，VSync作用是确保GPU每次只在显示器准备好接收新帧时才发送帧。它会限制GPU发送新帧的时间点，必须与显示器的刷新周期对齐。当GPU完成一帧的渲染时，它不会立刻将这帧送到显示器，而是等到显示器下一次刷新周期开始时，才发送这帧到显示器。这样可以保证显示器在一个刷新周期内只显示了一帧图像，避免了撕裂。
从Android角度来看，VSync需要做的事情是产生VSync信号，通知CPU/GPU立刻开始生成下一帧，通知GPU从FrameBuffer中奖当前帧post到显示器，即生成帧时机必须和Vsync信号保持同步。

### 3.3 VSync的限制
- 输入延迟：由于GPU需要等待显示器的刷新周期，这可能导致输入相应有稍微延迟。这对于游戏应用来说，影响比较严重。
- 帧率下降：如果GPU不能及时完成帧的渲染以匹配每个刷新周期，帧率可能会下降。

在当仅有一个FrameBuffer并且启用Vsync的配置，虽然可以避免屏幕撕裂来提升图像质量，但其也带来性能上的损失，如增加延迟，帧率不稳定等。多FrameBuffer技术的引入，能够有效缓解这些问题，提供更加优秀的体验。

## 四、多缓冲技术
### 4.1 双缓冲技术
系统使用两个FrameBuffer,FrontBuffer（屏幕上显示的帧）和BackBuffer（后台正在渲染的帧），同时结合VSync，可以有效避免屏幕撕裂的问题。
 - 基本流程：
    - BackBuffer进行全部的渲染操作
    - 在渲染完成后，整个BackBuffer会一次性提交到FrontBuffer，然后直接显示在屏幕上，这样用户看到是一个完整的数据。换句话说，在一帧被渲染完成后才会传递给屏幕，不会看到半成品的帧。
    - 在FrontBuffer展示的同时，BackBuffer继续进行下面的渲染操作
 - 问题：
    - 双缓存配合VSync能明显改善屏幕撕裂现象，但是这种组合在某些情况下会导致页面卡顿。在启动VSync的情况下，硬件个别时候资源紧张，导致后台绘制如果不能及时完成帧渲染，它就必须要等待下一个VSync信号才能将数据与FrontBuffer进行交换，此时仍旧显示前一个FrontBuffer,即造成卡顿。
### 4.2 三缓冲技术
在原有的基础上再添加一个后缓冲区，即一个前缓冲区，两个后缓冲区，这样可以进步一减少卡顿。
- 优化资源利用：
    - 在双缓冲系统中，如果后缓冲区正在渲染而前缓冲区正在显示，则GPU在完成渲染任务后必须等待下一个Vsync信号到来才能开始下一帧的渲染，这种等待导致GPU在某些时间段内处于空闲状态，未能充分利用其计算能力。
    - 三缓冲添加了第二个BackBuffer，此时当一个BackBuffer完成渲染并准备交换到FrontBuffer时，GPU可以立即在另外一个BackBuffer上渲染下一帧，无需等待当前帧显示完毕，减少了GPU的空闲时间。
- 问题：
    - 带来了更高的资源消耗
## 五、Anrdoid Choreographer
Android中的Choreographer可以视作Vsync信号与上层应用渲染之间的桥梁。它的主要作用是协调屏幕刷新率和应用渲染操作，确保UI的更新和屏幕的刷新过程能够同步进行，从而提高应用的表演和用户的体验。
### 5.1 源码流程
#### 5.1.1 Choreographer(Looper looper,int vsyncSource)
- 1、绑定Looper
- 2、创建FrameHandler，用于处理VSync信号、帧率计算、各种Callback回调等
- 3、创建FrameDisplayEventReceiver，用于接收Vsync信号
- 4、初始化CallBackQueues
```java
private Choreographer(Looper looper, int vsyncSource) {
        //1、绑定Looper
        mLooper = looper;
        //2、创建FrameHandler，用于处理VSync信号、帧率计算、各种Callback回调等
        mHandler = new FrameHandler(looper);
        //3、创建FrameDisplayEventReceiver，用于接收SurfaceFlinger发给应用的Vsync信号
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        mLastFrameTimeNanos = Long.MIN_VALUE;

        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
        
        //4、初始化CallBackQueues
        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
        // b/68769804: For low FPS experiments.
        setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
}
```

#### 5.1.2 FrameHandler 
- MSG_DO_FRAME:开始渲染下一帧
- MSG_DO_SCHEDULE_VSYNC：请求VSync信号
- MSG_DO_SCHEDULE_CALLBACK：处理各种Callback
```java
private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0, new DisplayEventReceiver.VsyncEventData());
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

#### 5.1.3 FrameDisplayEventReceiver 
继承自DisplayEventReceiver并实现了Runnable接口，Vsync信号的注册、申请、接收都是通过这个类。当前我们可以核心关注一下onVsync、run、scheduleVsync方法。
```java
        

private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
    /**
     * 负责接收VSync信号，并post到UI线程
     * */
    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
        VsyncEventData vsyncEventData) {
            //...省略
            mTimestampNanos = timestampNanos;
            mFrame = frame;
            mLastVsyncEventData = vsyncEventData;
            //收到VSync信号，将自身（Runnable）通过Message传递给FrameHandler,执行下面的run方法
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
            //...省略
        }

    /**
     * 参考5.1.4
     * */
    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
    }

    /**
     * 注册下一帧的回调，这个方法在父类DisplayEventReceiver，与native进行交互
     * 整体的处理流程可参考5.1.5
     * */
    public void scheduleVsync(){
        //...
        nativeScheduleVsync(mReceiverPtr);
    }

}
```
#### 5.1.4 Choreographer.doFrame
```java
void doFrame(long frameTimeNanos, int frame,
            DisplayEventReceiver.VsyncEventData vsyncEventData) {
    //...省略
    startNanos = System.nanoTime();
    //jitterNanos计算出实际开始处理帧的时间和预定处理帧的时间之间的差异。
    //可以理解为帧处理的延迟值。
    final long jitterNanos = startNanos - frameTimeNanos;
    //如果延迟值大于单帧时长  frameIntervalNanos = (long)(1000000000 / getRefreshRate())
    if (jitterNanos >= frameIntervalNanos) {
        long lastFrameOffset = 0;
        //计算得到丢帧数skippedFrames以及最后一帧的偏移量lastFrameOffset（后者应该为了让统计更加准确）
        lastFrameOffset = jitterNanos % frameIntervalNanos;
        final long skippedFrames = jitterNanos / frameIntervalNanos;

        //如果丢帧超过阈值，则进行打印
        //SKIPPED_FRAME_WARNING_LIMIT 来源自SystemProperties.getInt("debug.choreographer.skipwarning", 30);
        // debug.choreographer.skipwarning 这个值可以通过修改源码或者是Hook的手段进行处理。
        if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
            Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                                    + "The application may be doing too much work on its main "
                                    + "thread.");
            }
        if (DEBUG_JANK) {
            Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                                    + "which is more than the frame interval of "
                                    + (frameIntervalNanos * 0.000001f) + " ms!  "
                                    + "Skipping " + skippedFrames + " frames and setting frame "
                                    + "time to " + (lastFrameOffset * 0.000001f)
                                    + " ms in the past.");
        }
    }
    frameTimeNanos = startNanos - lastFrameOffset;
    frameData.updateFrameData(frameTimeNanos);

    //省略...

    //包含markInputHandlingStart、markAnimationsStart、markPerformTraversalsStart
    //穿插在下面doCallback之间，在对应的实际会做信息记录
    //使用adb dumpsys gfxinfo看到的信息就是这里记录的
    mFrameInfo.markXXX();

    //处理各种Callback，包含输入事件、动画、绘制等


    //INPUT事件经过处理后，最终会传递到DecorView.dispatchTouchEvent
    doCallbacks(Choreographer.CALLBACK_INPUT, frameData, frameIntervalNanos);

    //日常统计帧率使用的Choreographer.getInstance().postFrameCallback，就是将Callback计入到CALLBACK_ANIMATION的队列中，此时也就是Callbacl中doFrame方法的回调时机。
    doCallbacks(Choreographer.CALLBACK_ANIMATION, frameData, frameIntervalNanos);
    doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameData,
                    frameIntervalNanos);
    
    //ViewRootImpl中scheduleTraversals注册的就是CALLBACK_TRAVERSAL类型的callback
    //这个callback中执行就是ViewRootImpl.doTraversal()->performTraversals（）-> performMeasure() -> performLayout() -> performDraw()
    doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameData, frameIntervalNanos);
    doCallbacks(Choreographer.CALLBACK_COMMIT, frameData, frameIntervalNanos);
}

```

#### 5.1.5 FrameDisplayEventReceiver.scheduleVsync
当App在调用requestLayout、invalidate、setLayoutParams、动画等行为的时候，会执行到
Choregrapher.postCallback -> Choregrapher.postCallbackDelayedInternal -> Choregrapher.scheduleFrameLocked -> FrameDisplayEventReceiver.scheduleVsync() ,
进而请求Vsync信号。

### 5.2 Choreographer与同步屏障
#### 5.2.1 什么是同步屏障
同步屏障消息就是在消息队列中插入一个屏障，在屏障后面的消息都会被阻隔，不能被处理。不过异步消息除外，屏障不会阻挡异步消息。
#### 5.2.2 Choreographer与同步屏障
为了更快响应VSync信号以及渲染操作，在Choreographer执行相关操作的时候，一般伴随着同步屏障。以ViewRootImpl.scheduleTraversals为例，在其中请求Vsync信号的时刻，会先进行同步屏障。
```java
public final class ViewRootImpl{
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            //在队列中插入同步屏障
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            //移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
            //...
        }
    }
}
```
Choreographer中Handler相关的操作会发送异步消息，避免同步屏障的影响，更快地响应相关的操作。
```java
public final class Choreographer{

    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        
        //...
        Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
        msg.arg1 = callbackType;
        //异步消息
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, dueTime);
        //...
    }

    private void scheduleFrameLocked(long now) {
        //...
        Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
        //异步消息
        msg.setAsynchronous(true);
        mHandler.sendMessageAtFrontOfQueue(msg);
        //...
    }
}
```