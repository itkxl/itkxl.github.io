# 刷新率获取
# 帧率计算
# 丢帧计算
# 评分
# 优化点  

# 滑动速度？  手势触碰




# HZ 和 FPS的概念区别
# FrameBuffer 单个缓冲， 画面撕裂
# VSYNC & SurfaceFlinger
# 双重缓存&三重缓存



统计帧率的维度

    平均帧率：
        记录在一定时间段内的平均FPS值。这可以给出一个大致的性能指标，显示应用在正常条件下的运行情况。

    最低帧率：
        记录同一时间段内的最低FPS值。这对于识别可能的性能瓶颈点尤其重要，因为低帧率往往与用户体验下降直接相关。

    帧率波动：
        测量帧率的稳定性，即帧率随时间变化的程度。高波动可能意味着应用在处理不同的场景或是在不同的硬件上运行时表现不一致。

    分布统计：
        分析不同帧率区间的分布，例如多少百分比的时间帧率超过60FPS，多少时间低于30FPS等。这有助于了解应用在各种性能级别上的表现。

    按设备/操作系统版本：
        分析不同设备或操作系统版本上的FPS表现。这有助于识别特定设备或系统上可能存在的兼容性问题。

    按应用状态：
        比如在不同界面、不同功能模块的FPS表现。这可以帮助开发者了解哪些功能可能引起性能问题。




ublic class DynamicProxyDemo {

    public static void setDynamicWindowCallback(Window window) {
        final Window.Callback originalCallback = window.getCallback();

        Window.Callback dynamicProxy = (Window.Callback) Proxy.newProxyInstance(
                Window.Callback.class.getClassLoader(),
                new Class[]{Window.Callback.class},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        // 在调用任何原始方法之前可以插入自定义逻辑
                        logMethodAccess(method, args);  // 示例：记录方法调用

                        // 调用原始的方法
                        return method.invoke(originalCallback, args);
                    }
                });

        window.setCallback(dynamicProxy);
    }

    private static void logMethodAccess(Method method, Object[] args) {
        // 这里可以实现一些日志记录的功能
        System.out.println("Method called: " + method.getName());
    }
}




https://codefuturesql.top/post/android_get_frame_data/