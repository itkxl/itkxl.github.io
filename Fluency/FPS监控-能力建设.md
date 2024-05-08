## 一、页面帧率统计
### 1.1 帧率统计
在上一篇介绍帧率的基础知识的文章中，Choreographer收到VSync信号后，会依次调用Input,Animation,Draw等Callback，我们可以使用Choreographer.postFrameCallback往Chaoreographer中注册Callback的方式来统计帧率。在FrameCallback.doFrame（long frameTimeNanos）中可以得到Vsync的时间戳，两次回调的时间戳的差值可以近似上一帧的渲染耗时。
```java
public class FPSFrameCallback implements Choreographer.FrameCallback{

    private long mFirstFrameTime;
    private int mFrameCount;
    private static final long SECOND = TimeUnit.SECONDS.toNanos(1);

    //FPS集合，可用于计算平均帧率，最低帧率，帧率波动等信息。
    private List<Integer> mFrameList = new ArrayList<>();
    private boolean isRunning = true;

    @Override
    public void doFrame(long frameTimeNanos) {

        if (!isRunning){
            mFrameList.clear();
            return;
        }

        if (mFirstFrameTime == 0){
            mFirstFrameTime = frameTimeNanos;
        }
        //帧数累加
        mFrameCount ++;
        
        //时间累计超过1秒，计算总帧数
        long timeDiff = frameTimeNanos - mFirstFrameTime;
        if (timeDiff >= SECOND){
            mFrameList.add(mFrameCount);

            //重置
            mFrameCount = 0;
            mFirstFrameTime = frameTimeNanos;
        }

        Choreographer.getInstance().postFrameCallback(this);
    }

    public void destroy(){
        isRunning = false;
        mFrameList.clear();
    }
}
```
这是一个最基础的帧率计算的代码，在适当的实际进行调用并上报相关数据，可以得到一个基础的监控。  
但是其并相对比较简陋，需要逐步完善。
 - 由于不停地postFrameCallback，会导致即使没有绘制任务也会不停请求VSync信号，造成一定的性能损耗。
 - 仅能充当一个监控数据，无法究因。
### 1.2 绘制帧率统计
理论上，我们不需要一直监听帧率，如果可以，只需要监听绘制期间产生的帧率即可，如果页面配有绘制也一直进行统计，会拉高页面的帧率（不需要绘制的时候，页面的帧率是稳定的）。

如何监听页面正在绘制呢？需要使用到ViewTreeObserver相关的Api。

```java
//监听页面根View是否产生绘制行为
//此Api没有结束回调，可以post一个Runnable,x毫秒没有调用，则认为绘制结束。收集此时间段内的FPS,即可认为是绘制帧率
//addOnDrawListener回调相对比较频繁，也可以考虑使用addOnGlobalLayoutListener
activity.getWindow().getDecorView().getViewTreeObserver().addOnDrawListener(new ViewTreeObserver.OnDrawListener() {
    @Override
    public void onDraw() {
        //省略...

        if (!mPageDrawing){
            mPageDrawing = true;
        }
        //可以在mDrawStopRunnable中将标志位复原
        mPageActionDetectHandler.removeCallbacks(mDrawStopRunnable);
        mPageActionDetectHandler.postDelayed(mDrawStopRunnable, defaultDelayTime);
    }
});
```

### 1.3 滑动帧率统计
列表滑动是否顺畅，是用户体验一个非常重要的指标，故需要将滑动的帧率进行统计。此处依旧要使用到ViewTreeObserver相关的Api。

```java
//此处与addOnDrawListener类似，依旧没有停止滑动的直接回调，需要参考onDraw中的处理做延迟判定
activity.getWindow().getDecorView().getViewTreeObserver().addOnScrollChangedListener(new ViewTreeObserver.OnScrollChangedListener() {
    @Override
    public void onScrollChanged() {
        //省略...

        if(!mPageScrolling){
            mPageScrolling = true;
        }
        //
        mPageActionDetectHandler.removeCallbacks(mScrollingStopRunnable);
        mPageActionDetectHandler.postDelayed(mScrollingStopRunnable, defaultDelayTime);
    }
});
```
此时可以统计到页面滑动所生成的帧率，但是很快就发现了新的问题，页面自动滑动的广告Banner也会触发这个回调，导致非用户操作的滑动帧率也被统计进来，接下来介绍用户手动滑动帧率统计。
### 1.4 用户手动滑动帧率统计
想要统计到用户手动滑动帧率，需要拿到用户在当前页面的操作手势。
#### 1.4.1 通过在DevorView下插入一个View来捕获手势
```java
    application.registerActivityLifecycleCallbacks(new AbsActivityLifecycle() {
            @Override
            public void onActivityResumed(@NonNull Activity activity) {
                View view = activity.getWindow().getDecorView();
                if (!(view instanceof ViewGroup)) {
                    return;
                }
                ViewGroup root = (ViewGroup) view;
                if (root.getChildCount() <= 0) {
                    return;
                }
                View child = root.getChildAt(0);
                if (child instanceof TouchInterceptor) {
                    return;
                }
                //在中间插入自定义的View-TouchInterceptor,用于拦截事件进行分发
                root.removeView(child);
                TouchInterceptor touchInterceptor = new TouchInterceptor(activity);
                root.addView(touchInterceptor, new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
                touchInterceptor.addView(child, new TouchInterceptor.LayoutParams(child.getLayoutParams()));
            }
    });


    private static class TouchInterceptor extends FrameLayout {

        //省略...

        @Override
        public boolean dispatchTouchEvent(MotionEvent event) {
            //将用户手势进行分发。
            getInstance().dispatchEventToHandler(event.getAction(), event.getX(), event.getY());
            return super.dispatchTouchEvent(event);
        }

        //...
    }
```
#### 1.4.2 动态代理Window.Callback
##### 1.4.2.1 Activity中事件传递的流程
- Activity与Window.Callback
    - 透过Activity的源码可知，其实现了‘Window.Callback’接口，这使得Acitivyt可以响应windows的各种事件。
- 设置回调
    - Activity的onCreate中方法内，Activity通过‘getWindow().setCallback(this)’将自己作为回调传递给window。
- 事件的传递
    - 当触摸事件发生时，Android系统将事件传递给Window,然后Window通过其'disPatchTouchEvent'来处理这个事件。
    - 同时也就走到了Activity这个回调，Activity再将其分发给应用的视图。

##### 1.4.2.2 动态代理
由于Window.Callback是一个接口，可以通过动态代理的方式进行拦截并处理。下面是一个Demo
```java
public void setDynamicWindowCallback(Window window) {
    final Window.Callback originalCallback = window.getCallback();
    Window.Callback dynamicProxy = (Window.Callback) Proxy.newProxyInstance(
                Window.Callback.class.getClassLoader(),
                new Class[]{Window.Callback.class},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        // 在调用任何原始方法之前可以插入自定义逻辑
                        reportAction(method, args);  // 示例：记录方法调用

                        // 调用原始的方法
                        return method.invoke(originalCallback, args);
                    }
                });
        window.setCallback(dynamicProxy);
    }

    private void reportAction(Method method, Object[] args) {
        // 这里可以实现记录或者是打标
    }
```

### 1.5 页面刷新率获取
前面花了一些篇幅介绍如何获取各种情况的帧率情况，这些数据还需要屏幕刷新率作参考才有意义，在不同的屏幕刷新率下统计对应的帧率才有意义，下面介绍一下刷新率的获取。
```java
/**
 * 获取当前的刷新率
 * */
private float getRefreshRate(Activity activity){
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        return activity.getDisplay().getRefreshRate();
    }
    return getWindow().getWindowManager().getDefaultDisplay().getRefreshRate();
}


/**
 * 监听页面刷新率实时变化
 */
DisplayManager displayManager = (DisplayManager) getSystemService(DISPLAY_SERVICE);
displayManager.registerDisplayListener(new DisplayManager.DisplayListener() {
        @Override
        public void onDisplayAdded(int displayId) {

        }

        @Override
        public void onDisplayRemoved(int displayId) {

        }

        @Override
        public void onDisplayChanged(int displayId) {
            Display display = displayManager.getDisplay(displayId);
            if (display != null){
                //获取改变后的刷新率
                display.getRefreshRate();
            }
        }
},null);
```

## 二、数据统计维度

### 2.1 平均帧率和最低帧率
这两项属于基础数据
- 平均帧率：记录在一定时间段内的平均FPS值。这可以给出一个大致的性能指标，显示应用在正常条件下页面的流畅情况。使用前面的集合中的数据直接就求平均值即可。
- 最低帧率：记录同一时间段内的最低FPS值。这对于识别页面流畅相对比较重要，因为低帧率往往与用户体验下降直接相关。使用前面的集合中的数据求最小值即可。
  
### 2.2 帧率波动
帧率波动是评价页面是否流畅的一个非常重要的指标。一个页面如果长时间维持在一个同一个帧率下，用户的体感不一定会觉得卡顿，但是如果帧率波动非常厉害，那么用户的卡顿感就会比较强烈，这个情况单单依靠平均帧率是无法得到有效评估的。
假设有以下两组帧率数据
[50,50,50,50,50]
[60,40,60,40,50]
虽然两者计算得到的平局值是一样的，但是后者用户的体感一定是不如前者的。
如何体现这个帧率波动呢？

#### 2.2.1 标准差计算
使用标准差是一个比较好的衡量，标准差越大，表明帧率波动越大。
还是以上面两组数据举例
- 对于第一组数据 [50,50,50,50,50][50,50,50,50,50]，标准差为 0，表示没有波动。
- 对于第二组数据 [60,40,60,40,50][60,40,60,40,50]，标准差约为 8.94，表示数据相对于平均值不小的波动。
  
```java
/**
 * 标准差计算流程
 * 1、计算平均值
 * 2、计算每个数据点与平均值的差的平方
 * 3、计算平方差的平均值（方差）
 * 4、求平方根得到标准差
 * */
public class StandardDeviationCalculator {

    // 计算标准差
    public static double calculateStandardDeviation(int[] numbers) {
        if (numbers == null || numbers.length == 0) {
            throw new IllegalArgumentException("The array cannot be null or empty.");
        }

        double mean = calculateMean(numbers);
        double variance = calculateVariance(numbers, mean);
        return Math.sqrt(variance);
    }

    // 计算平均值
    private static double calculateMean(int[] numbers) {
        double sum = 0;
        for (int num : numbers) {
            sum += num;
        }
        return sum / numbers.length;
    }

    // 计算方差
    private static double calculateVariance(int[] numbers, double mean) {
        double sumOfSquaredDifferences = 0;
        for (int num : numbers) {
            sumOfSquaredDifferences += Math.pow(num - mean, 2);
        }
        return sumOfSquaredDifferences / numbers.length;
    }
}
```


#### 2.2.2 丢帧评分
整体思路与标准差计算类似，不过这里是使用丢帧来作为计算因子，前面已经介绍过获取实时刷新率的方式，以实时刷新率-帧率可以得到丢帧数。

**评分 =  丢帧数*单帧时常（整数）*2的丢帧次数方*发生次数/对应时长**

```java

    /**
     * fpsInfoBuckets:丢失的帧数，以及对应丢失帧数锁发生的次数，使用Map来进行描述，key为丢失的帧数，value为发生的次数
     * ratetime:单帧市场
     * duration：持续时间
     * */
    private float calculateScore(Map<Integer, Integer> fpsInfoBuckets, int ratetime,long duration) {
        int totalScore = 0;
        for (Map.Entry<Integer, Integer> entry : fpsInfoBuckets.entrySet()) {
            int missedFrames = entry.getKey();
            int occurrences = entry.getValue();
            int missedFramesSection = getSectionForMissedFrames(missedFrames);
            totalScore += missedFramesSection * ratetime * Math.pow(2, missedFramesSection) * occurrences;
        }
        return (float) (totalScore * 100.0 / duration);
    }

    /**
     * 丢失帧数过多，可能导致算得的分数过大，故此处将其改为一个指定的系数
     * */
    private int getSectionForMissedFrames(int missedFrames) {
        if (missedFrames <= 0) {
            return 0;
        } else if (missedFrames <= 2) {
            return 1;
        } else if (missedFrames <= 6) {
            return 2;
        } else if (missedFrames <= 13) {
            return 3;
        } else if (missedFrames <= 24) {
            return 4;
        } else {
            return 5;
        }
    }
```
### 2.3 分布统计
这个主要是在云端做数据聚合的时候需要考虑的。帧数、以及评分等都要分析不同区间的占比。例如多少百分比的时间帧率超过60FPS，多少时间低于30FPS等。这有助于了解应用在各种性能级别上的表现。


### 2.4 其他维度
- 设备基本信息
- App版本信息
- 设备分级信息
- 页面信息
- 核心业务&模块信息
- ...


## 三、踩坑
- getDecorView()如果调用时机过早，在6.0即一下版本可能导致偶先崩溃
- Choreographer.postFrameCallback如果没有做上述优化，调用频繁，在小米14上偶先崩溃
