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
    - 定义：单位时间内&单次进程内存使用峰值
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

## 六、KOOM原理解析
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

### 6.1 DumpHprof进程冻结问题
#### 6.1.1 产生原因
由于DumpHprof的时候，需要保证内存无变化，需要将整个虚拟机所有线程都暂停掉，外加dump时间较长，从而造成应用冻结。
```cpp
/**
 * 位置：art/runtime/hprof/hprof.cc
 **/
void DumpHeap(const char* filename, int fd, bool direct_to_ddms) {
  CHECK(filename != nullptr);
  Thread* self = Thread::Current();
  gc::ScopedGCCriticalSection gcs(self,
                                  gc::kGcCauseHprof,
                                  gc::kCollectorTypeHprof);

  //暂停掉所有线程
  ScopedSuspendAll ssa(__FUNCTION__, true /* long suspend */);
  Hprof hprof(filename, fd, direct_to_ddms);
  hprof.Dump();
}

/**
 * 位置：art/runtime/thread_list.cc
 * 在DumpHeap中调用此方法时候，可以看到仅调用了SuspendAll，并没有执行ResumeAll
 * 这里是利用了C++的RAII资源管理方式，在{}结束，自动调用析构函数达到执行ResumeAll的目的
 * 故在执行完Dump后，会回复线程调用
 **/
ScopedSuspendAll::ScopedSuspendAll(const char* cause, bool long_suspend) {
  Runtime::Current()->GetThreadList()->SuspendAll(cause, long_suspend);
}

ScopedSuspendAll::~ScopedSuspendAll() {
  Runtime::Current()->GetThreadList()->ResumeAll();
}

```

#### 6.1.2 进程冻结解决方案
从6.1.1的源码来看子线程是无法解决冻结问题的，这里KOOM采用了Fork子进程的方式来处理，由于Linux在Fork子进程的“写时复制机制”，fork的那一瞬间，其内存布局与父进程是共用的（并不是单独copy一份），当父进程或者是子进程在这之后发生内存变更，系统才会分配新的内存。

既然如此，是否我们在执行fork之后，在子进程做dumpHprof是否可以解决问题？实践下载是不行的，代码会在ScopedSuspendAll挺住，因为自己成在执行ScopedSuspendAll的时候等不到其他线程执行到chekpooint返回暂停结果的。

KOOM的操作是在主进程先执行SuspendAll,使ThreadList中保存的所有的线程状态为suspend，之后进行fork,子进程共享父进程的ThreadList全局变量list_,可以欺骗虚拟机，使其认为全部线程已经完成了暂停操作，进而可以执行dump Hprof行为。此时主进程可以直接执行ResumeAll恢复行为。

```java
@Override
  public synchronized boolean dump(String path) {
    // 省略
    boolean dumpRes = false;
    try {
      //暂停当前进程&Fork子进程，fork方法会返回两次结果
      int pid = suspendAndFork();
      // pidf返回0，表示子进程，可以执行dump操作
      if (pid == 0) {
        Debug.dumpHprofData(path);
        exitProcess();
      } else if (pid > 0) {
        //pid大于0，返回主进程，此时执行resume
        dumpRes = resumeAndWait(pid);
      }
    } catch (IOException e) {
    }
    return dumpRes;
  }
```

### 6.2 Hprof裁剪
裁剪有两种方式
- Dump完成后对文件进行裁剪
- Dump的同时进行实时裁剪

KOOM和Tailor使用的都是第二种方式，对IO相关的Api进行PLT Hook，在真正写入之前完成裁剪，减少用户磁盘占用以及上传使用的流量。

#### 6.2.1 Hook IO行为
在Dump之前开启Hook，Dump结束后可以将这部分Hook关闭。
```cpp
  /**
   *
   * android 7.x，write方法在libc.so中
   * android 8-9，write方法在libart.so中
   * android 10，write方法在libartbase.so中
   * libbase.so是一个保险操作，防止前面2个so里面都hook不到(:
   *
   * android 7-10版本，open方法都在libart.so中
   * libbase.so与libartbase.so，为保险操作
   */
  xhook_register("libart.so", "open", (void *)HookOpen, nullptr);
  xhook_register("libbase.so", "open", (void *)HookOpen, nullptr);
  xhook_register("libartbase.so", "open", (void *)HookOpen, nullptr);

  xhook_register("libc.so", "write", (void *)HookWrite, nullptr);
  xhook_register("libart.so", "write", (void *)HookWrite, nullptr);
  xhook_register("libbase.so", "write", (void *)HookWrite, nullptr);
  xhook_register("libartbase.so", "write", (void *)HookWrite, nullptr);


  /**
   * HookOpen最终会走到HookOpenInternal方法中
   * 内部判断如果此时打开的文件路径与我们制定的Hprof输出路径一致，则记录文件描述符fd,以及状态，为后续的HookWrite拦截做准确。
   **/
  int HprofStrip::HookOpenInternal(const char *path_name, int flags, ...) {
    va_list ap;
    va_start(ap, flags);
    int fd = open(path_name, flags, ap);
    va_end(ap);

    if (hprof_name_.empty()) {
        return fd;
    }

    //根据路径判断是否为Hprof的输出路径
    if (path_name != nullptr && strstr(path_name, hprof_name_.c_str())) {
    hprof_fd_ = fd;
        is_hook_success_ = true;
    }
    return fd;
}

```

#### 6.2.2 实时裁剪
Hprof主要以Header和Record组成，Header记录Hprof的整体信息，Record则分很多类型，每种类型有单独的TAG进行标识。内存中大部分数据集中在PRIMITIVE ARRAY DUMP，但其中不包括对象的大小以及引用关系，故这部分是裁剪的重点。
##### 部分核心裁剪逻辑
 - 裁剪HPROF_PRIMITIVE_ARRAY_DUMP
```cpp
case HPROF_PRIMITIVE_ARRAY_DUMP: {
      int primitive_array_dump_index = first_index + HEAP_TAG_BYTE_SIZE /*tag*/
                                       + OBJECT_ID_BYTE_SIZE +
                                       STACK_TRACE_SERIAL_NUMBER_BYTE_SIZE;
      int length =
          GetIntFromBytes((unsigned char *)buf, primitive_array_dump_index);
      primitive_array_dump_index += U4 /*Length*/;

      // 裁剪掉基本类型数组，无论是否在system space都进行裁剪
      // 区别是数组左坐标，app space时带数组元信息（类型、长度）方便回填
      if (is_current_system_heap_) {
        strip_index_list_pair_[strip_index_ * 2] = first_index;
      } else {
        strip_index_list_pair_[strip_index_ * 2] =
            primitive_array_dump_index + BASIC_TYPE_BYTE_SIZE /*value type*/;
      }
      array_serial_no++;

      int value_size = GetByteSizeFromType(
          ((unsigned char *)buf)[primitive_array_dump_index]);
      primitive_array_dump_index +=
          BASIC_TYPE_BYTE_SIZE /*value type*/ + value_size * length;

      // 数组右坐标
      strip_index_list_pair_[strip_index_ * 2 + 1] = primitive_array_dump_index;

      // app space时，不修改长度因为回填数组时会补齐
      if (is_current_system_heap_) {
        strip_bytes_sum_ += primitive_array_dump_index - first_index;
      }
      strip_index_++;

      array_serial_no = ProcessHeap(buf, primitive_array_dump_index, max_len,
                                    heap_serial_no, array_serial_no);
    } break;
```

- 裁剪system space（即Zygote和Image）
```cpp
case HPROF_INSTANCE_DUMP: {
      int instance_dump_index =
          first_index + HEAP_TAG_BYTE_SIZE + OBJECT_ID_BYTE_SIZE +
          STACK_TRACE_SERIAL_NUMBER_BYTE_SIZE + CLASS_ID_BYTE_SIZE;
      int instance_size =
          GetIntFromBytes((unsigned char *)buf, instance_dump_index);

      // 裁剪掉system space
      if (is_current_system_heap_) {
        strip_index_list_pair_[strip_index_ * 2] = first_index;
        strip_index_list_pair_[strip_index_ * 2 + 1] =
            instance_dump_index + U4 /*占位*/ + instance_size;
        strip_index_++;

        strip_bytes_sum_ +=
            instance_dump_index + U4 /*占位*/ + instance_size - first_index;
      }

      array_serial_no =
          ProcessHeap(buf, instance_dump_index + U4 /*占位*/ + instance_size,
                      max_len, heap_serial_no, array_serial_no);
} break;
```

此处就暂且省略Tailor的源码解析了，两者整体原理类似，KOOM裁剪更加充分，Tailor保留了System space,可以更加全面的分析内存问题。


## 七、补充
- 在上线这部分的时候，我们当时的紧急问题是部分用户在短时间内(2个小时左右)就发生了OOM,为了减少对用户的二次伤害，故内存泄漏的分析没有使用端上解析能力，而是上传到服务端后进行解析。端上解析能力除了KOOM开源的对shark的优化版本外，可以关心一个Matrix近期更新的C/C++版本的解析组件。
- 端上解析能力理论上没有必要采用全量解析，只需要抓住几个我们核心的组件或者是页面做重点分析即可，可以省时省力，减少对用户的影响。
- 在观察Hprof文件的时候，不能单单关心内存泄漏，尤其是首页，它一般是不会被关闭的，也就意味着它不存在狭义上的内存泄漏，但是如果使用对象没有得到合理的释放并逐步增加，也会对内存造成巨大问题。
- Hprof除了通过监控用户内存水位或者是异常的时候被动Dump&上传外，我们在分析一些奇怪的问题的时候，也可以通过接口&长连接主动触发用户的Dump行为，回捞后进行单点分析。