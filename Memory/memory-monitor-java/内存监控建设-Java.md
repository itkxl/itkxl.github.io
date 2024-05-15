## 一、背景
内存是内容的载体，许多用户看到或看不到的问题在一定程度上都可以与内存问题相关。比如，应用程序的卡顿、崩溃、加载缓慢等现象，往往是由于内存不足、内存泄漏或内存管理不当引起的。而用户看不到的后台进程问题、数据处理错误等，也可能源自内存的使用效率和分配策略不当。

在Android平台上，ART（Android Runtime）虚拟机负责应用的运行和内存管理。随着应用的复杂性和数据量的增加，内存问题变得更加突出。为了确保应用稳定性，监控ART虚拟机的内存使用情况变得尤为重要。
## 二、目标
- 内存监控数据的收集与分析
- 建立内存监控指标体系
- 建设线上内存问题排查工具
## 三、Java内存数据采集
### 3.1 虚拟机内存使用情况
- Heap Size（堆大小）
    - 定义：Java堆的总大小，是JVM为应用程序分配的内存区域，用于存储对象实例。
    - 获取方式：Runtime.getRuntime().maxMemory()
- Allocated Memory(已分配内存)
    - 定义：当前已经分配并使用的内存
    - 获取方式：Runtime.getRuntime().totalMemory()
- Free Memory (可用内存)
    - 定义：堆中未分配的内存量。
    - 获取方式：Runtime.getRuntime().freeMemory()
- Peak Memory Usage (内存峰值)
    - 定义：单位时间内内存使用峰值
    - 获取方式：通过对Allocated Memory的聚合&计算获取

### 3.2 GC
#### 3.2.1 基本概念
Android ART GC分为阻塞式GC和非阻塞式GC.
- 阻塞式GC:
    - 特点：
        - 暂停应用：阻塞式回收会暂停应用的所有线程，造成应用短暂的卡顿。
        - 全局回收：阻塞式回收会将整个虚拟机内全范围的扫描与回收。
    - 优点：简单、效率高
    - 缺点：短暂卡顿，用户有感。
- 非阻塞式GC：
    - 特点：
        - 并发回收：不阻塞线程，用户体验更佳。
        - 增量回收：可以分多次、逐步进行垃圾回收，避免一次行回收带来长时间卡顿。
    - 优点：减少了应用停顿时间，用户体验好。
    - 缺点：实现复杂，效率低。

#### 3.2.2 GC数据采集
- GC Events (GC 事件)
    - 定义：垃圾回收的次数
    - 获取方式：
        - 非阻塞式：Debug.getRuntimeStat("art.gc.gc-count")
        - 阻塞式：Debug.getRuntimeStat("art.gc.blocking-gc-count")
- GC Pause Time (GC 暂停时间)
    - 定义：垃圾回收占用的时长
    - 获取方式：
        - 非阻塞式：Debug.getRuntimeStat("art.gc.gc-time")
        - 阻塞式：Debug.getRuntimeStat("art.gc.blocking-gc-time")

### 3.3 OOM
#### 3.3.1 OOM的判定

```cpp
/**
 * 此段代码位于art/runtime/gc/heap-inl.h
 * allocator_type:内存分配器类型
 * alloc_size:请求分配的内存大小
 * grow:是否允许堆大小扩充
 **/
inline bool Heap::IsOutOfMemoryOnAllocation([[maybe_unused]] AllocatorType allocator_type,
                                            size_t alloc_size,
                                            bool grow) {
  //获取当前目标堆的大小
  size_t old_target = target_footprint_.load(std::memory_order_relaxed);
  while (true) {
    //获取当前已分配的内存大小
    size_t old_allocated = num_bytes_allocated_.load(std::memory_order_relaxed);
    //计算新分配内存后需要的内存大小
    size_t new_footprint = old_allocated + alloc_size;
    //检查新的内存占用是否在目标堆大小之内，如果是则返回false（没有OOM）。
    if (UNLIKELY(new_footprint <= old_target)) {
      return false;
    } 
    //检查新的内存占用是否超过增长限制（growth_limit_），如果是则返回true（出现OOM）
    else if (UNLIKELY(new_footprint > growth_limit_)) {
      return true;
    }
    // 如果当前正在进行并发垃圾回收（GC），则返回false（不判定为OOM，因为GC可能会释放内存）。
    if (IsGcConcurrent()) {
      return false;
    } else {
      //如果不在进行并发GC，并且允许增长目标堆大小：
      if (grow) {
        //更改当前堆的大小并记录
        if (target_footprint_.compare_exchange_weak(/*inout ref*/old_target, new_footprint,
                                                    std::memory_order_relaxed)) {
          VlogHeapGrowth(old_target, new_footprint, alloc_size);
          return false;
        }  // else try again.
      } else {
        return true;
      }
    }
  }
}
```
#### 3.3.2 OOM数据采集
与采集Crash一致，借助Thread.UncaughtExceptionHandler
```java
Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
    @Override
    public void uncaughtException(Thread thread, Throwable throwable) {
        if (throwable instanceof OutOfMemoryError) {
            //发生了OOM
        }
        //省略...
    }
);
```


### 3.4 Low Memory Killer
Low Memory Killer（LMK）是Android系统中的一个内存管理机制，用于在设备内存不足时自动终止一些后台进程，以释放内存并保持系统的稳定性和响应速度。LMK通常在设备内存压力较大时触发，通过杀掉优先级较低的进程，确保前台应用和系统关键进程能够继续运行。

### 3.4.1 LMK相关的参数
- oom_adj:
    - 路径：/proc/pid/oom_adj
    - 范围：-17至15
    - 作用：此值影响进程OOM的优先级，值越高表示进程在内存不足的时候越容易被杀死。
- oom_score:
    - 路径：/proc/pid/oom_score
    - 范围：0-1000
    - 作用：内核计算的值，用于表示在系统内存不足时被杀死的可能性。值越高，越容易被杀死。
- oom_score_adj
    - 路径：/proc/pid/oom_score_adj
    - 范围：-1000至1000
    - 作用：新的OOM调整至，用于取代‘oom_adj’，其范围更大，-1000表示进程绝对不会被杀死（比如Zygote就是-1090），1000表示进程最容易被杀死。
- oom_score&oom_score_adj的关系：后者是前者的计算因子。

### 3.4.2 LMK次数获取
获取LMK的次数，需要借助Android API 30提供的方法ActivityManager.getHistoricalProcessExitReasons
- 方法签名
```java
public List<ApplicationExitInfo> getHistoricalProcessExitReasons(String packageName, int pid, int maxNum);
```
- 返回值：ApplicationExitInfo
    - getReason()：返回退出原因的标志，可能的值包括：
        - ApplicationExitInfo.REASON_ANR：应用程序无响应 (ANR)。
        - ApplicationExitInfo.REASON_CRASH：应用程序崩溃。
        - ApplicationExitInfo.REASON_EXIT_SELF：应用程序自行退出。
        - ApplicationExitInfo.REASON_LOW_MEMORY：低内存导致的进程终止。
        - ApplicationExitInfo.REASON_SIGNALED：通过信号终止。
        - ApplicationExitInfo.REASON_OTHER：其他原因。
    - getTimestamp()：进程退出的时间戳。
    - getPid()：进程ID。
    - getPss()：进程的峰值常驻集大小（RSS，Resident Set Size），表示进程退出时使用的内存量。
    - getRss()：进程的峰值虚拟内存大小（Virtual Memory Size）。


其中核心项为REASON_LOW_MEMORY，同时可将Pss&Rss上报
## 四、Java内存指标建设
### 4.1 用户内存使用情况指标
- 用户内存水位变化
    - 定义：在用户使用过程中，每隔x分钟，用户使用内存百分比变化趋势。
    - 作用：可以整体了解App内存使用情况，根据App版本、ABTest等可观测更多维度的趋势
- 页面内存使用情况
    - 定义：页面可见、不可见 & 进入、退出内存变化
    - 作用：了解页面不同版本&ABTest下内存使用情况，其中可见与不可见目前客户端取得是多次均值。
- 用户GC情况
    - 定义：在用户使用过程中，每隔x分钟，用户GC的次数与耗时
    - 作用：了解页面不同版本&ABTest下App GC的情况，侧面反应内存问题。
### 4.2 用户异常指标
- OOM次数
    - 定义：单位时间内，圈定用户群 OOM总次数、OOM率
    - 作用：体现内存使用问题的直观指标。
- 高内存水位占比
    - 定义：单位时间内，圈定用户群内存使用超60%、80%次数
    - 作用：体现内存高危使用情况占比，也是优化措施上线后核心观测指标之一。
- LMK次数
    - 定义：单位时间内，圈定用户群App LMK总次数
    - 作用：一定程度上也能反映App内存整体使用情况。


## 五、Hprof文件
### 5.1 虚拟机内存问题大杀器-Hprof
Hprof文件是分析Java虚拟机内存的重要工具。它记录了Java堆的快照，详细展示了对象的内存分配情况。通过分析Hprof文件，开发者可以识别内存泄漏、优化内存使用、发现不必要的对象持有，以及理解对象间的引用关系。这些信息对于优化应用性能、提高内存利用效率至关重要。此外，Hprof文件还可以帮助定位内存消耗热点，找出可能导致内存溢出（OOM）的根本原因，从而确保应用的稳定性和可靠性。

### 5.2 线下采集Hprof方式
- 终端命令
```bash
adb shell am dumpheap <package_name> /data/local/heap_dump.hprof
```
- 利用Android Studio的Profile工具
- 系统Api
```java
Debug.dumpHprofData(filePath);
```

### 5.3 线上采集Hprof难点
- Debug.dumpHprofData会造成应用冻结
- Hprof文件过大（内存有大多，此文件就有多大），占用用户存储空间
- Hprof文件过大，上传对流量消耗有压力
- 原始的Hprof文件内部包含所有的用户数据，直接上传有隐私合规的风险

## 六、KOOM&Tailor 原理解析
为了解决5.3所概述的问题，采用的类Koom&Tailor的解决方案。
- KOOM
    - 地址：https://github.com/KwaiAppTeam/KOOM
    - 功能：关于JavaHprof文件相关内容，集子进程Dump、裁剪、本地解析为一体。
- Tailor:
    - 地址：https://github.com/bytedance/tailor
    - 功能：仅提供Hprof文件裁剪能力。
- 区别：
    - 整体功能：KOOM更加全面，比Tailor多了子进程Dumo&本地解析相关的能力
    - 裁剪：两者都具有裁剪功能，KOOM裁剪更加彻底，只保留了app-heap

### 6.1 DumpHprof卡顿问题
- 原因
由于DumpHprof的时候，需要保证内存无变化，需要将整个虚拟机所有线程都暂停掉，从而造成应用冻结。
- 解决方案
    - 失败：new一个子线程来DumpHprof，并不能解决这个问题，因为其是将所有的线程都暂停掉，自然也包含new出来的子线程
    - 成功：利用Linux的“写时复制”机制，fork一个子进程，此时其内存布局与原进程是完全一致的，然后进行dumpHprof,此时对就基本无影响了。
- 暂停虚拟机处理
```cpp

```