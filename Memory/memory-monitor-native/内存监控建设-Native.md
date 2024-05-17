## 一、背景
随着业务场景逐渐复杂，开发团队越来越依赖于原生代码（如C和C++）来提高性能和优化资源使用。跨端&业务&三方等场景Native代码越来越多，建设Native层的内存监控非常重要。
## 二、目标
- 提供Native内存使用数据：让开发者能够实时了解应用的内存使用情况，包括总内存使用量、峰值使用量等。
- 能及时发现线上出现的Native内存问题：内存泄漏&内存不合理使用
## 三、数据采集
这部分可以先看一下之前写的[内存映射文件概述](/Memory/memory-basic-knowledge/内存映射文件概述)

### 3.1 命令行获取
```bash
adb shell dumpsys meminfo pid
adb shell dumpsys meminfo packageName
```

以com.android.settings为例
```bash
itkxl@itkxl-Ubuntu:~$ adb shell dumpsys meminfo 5865
Applications Memory Usage (in Kilobytes):
Uptime: 219723726 Realtime: 584341125

** MEMINFO in pid 5865 [com.android.settings] **
                   Pss  Private  Private  SwapPss      Rss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty    Total     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------   ------
  Native Heap      737      696        0    37029     3608    56748    34496    17629
  Dalvik Heap      509      376        0    25311     7424    50963    12741    38222
 Dalvik Other     1350      144        0     3268     3084                           
        Stack      372      372        0     1008      380                           
       Ashmem       13        0        0        0      788                           
    Other dev       21        0       20        0      292                           
     .so mmap     2712      112       36       60    59264                           
    .jar mmap     1843        0        0        0    38588                           
    .apk mmap    16633        0    14644        0    20132                           
    .ttf mmap      200        0       84        0      704                           
    .dex mmap      792        4      780       12     1532                           
    .oat mmap      423        0        0        0    13568                         
    .art mmap      690      216        0     1325    25536                           
   Other mmap     1358        8      624        0     5640                           
      Unknown       96       88        0      592      952                           
        TOTAL    96354     2016    16188    68605   181492   107711    47237    55851
 
 App Summary
                       Pss(KB)                        Rss(KB)
                        ------                         ------
           Java Heap:      592                          32960
         Native Heap:      696                           3608
                Code:    15688                         136224
               Stack:      372                            380
            Graphics:        0                              0
       Private Other:      856
              System:    78150
             Unknown:                                    8320
 
           TOTAL PSS:    96354            TOTAL RSS:   181492       TOTAL SWAP PSS:    68605
 
 Objects
               Views:      819         ViewRootImpl:        4
         AppContexts:       28           Activities:        8
              Assets:       38        AssetManagers:        0
       Local Binders:       68        Proxy Binders:      117
       Parcel memory:     6196         Parcel count:      200
    Death Recipients:        4             WebViews:        0
 
 SQL
         MEMORY_USED:     2837
  PAGECACHE_OVERFLOW:     2551          MALLOC_SIZE:       46
 
 DATABASES
      pgsz     dbsz   Lookaside(b) cache hits cache misses cache size  Dbname
PER CONNECTION STATS
         4       28             21     5    16     2  /data/user_de/0/com.android.settings/databases/battery_settings.db
         4       48            122    42    70    14  :memory:
         4        8                    0     0     0    (attached) temp
         4     2244             74 14671 19706     9  /data/user_de/0/com.android.settings/databases/battery-usage-db-v8
         4        8                    0     0     0    (attached) temp
         4     2244             46   114    15     3  /data/user_de/0/com.android.settings/databases/battery-usage-db-v8 (2)
POOL STATS
     cache hits  cache misses    cache size  Dbname
              5            17            22  /data/user_de/0/com.android.settings/databases/battery_settings.db
             42            73           115  :memory:
          14787         19741         34528  /data/user_de/0/com.android.settings/databases/battery-usage-db-v8

```
这里看到的信息已经比较完善了，包含了Java堆、Native堆、mmap文件内存映射虚拟内存、物理内存等占用数据。
同时还提供了Java虚拟集中对象、binder、webView的数量以及db文件的使用情况。
通过这些数据可以在线下对目前App的内存使用情况有一个整体的了解。

同时也可以通过
```
cat /proc/pid/status    //虚拟内存、线程数、fd数量概览信息
cat /proc/pid/maps      //虚拟额存的详细数据
cat /proc/pid/smaps     //比maps数据更加详细
```
这部分可以先看一下之前写的[内存映射文件概述](/Memory/memory-basic-knowledge/内存映射文件概述),其中有更加详细的介绍。

### 3.2 代码获取
```java
Debug.MemoryInfo memoryInfo = new Debug.MemoryInfo();
Debug.getMemoryInfo(memoryInfo);

//Map中存储了内存相关的数据
Map<String, String> memoryStats = memoryInfo.getMemoryStats();


/**
 * 位置：frameworks/base/core/java/android/os/Debug.java
 **/
public Map<String, String> getMemoryStats() {
    Map<String, String> stats = new HashMap<String, String>();
    stats.put("summary.java-heap", Integer.toString(getSummaryJavaHeap()));
    stats.put("summary.native-heap", Integer.toString(getSummaryNativeHeap()));
    stats.put("summary.code", Integer.toString(getSummaryCode()));
    stats.put("summary.stack", Integer.toString(getSummaryStack()));
    stats.put("summary.graphics", Integer.toString(getSummaryGraphics()));
    stats.put("summary.private-other", Integer.toString(getSummaryPrivateOther()));
    stats.put("summary.system", Integer.toString(getSummarySystem()));
    stats.put("summary.total-pss", Integer.toString(getSummaryTotalPss()));
    stats.put("summary.total-swap", Integer.toString(getSummaryTotalSwap()));
    return stats;
}

```
- summary.native-heap：Native堆内存，由C/C++代码分配的内存总量；
- summary.code：包括应用的DEX代码、已优化或编译的ODEX代码、以及共享库所占用的内存；
- summary.stack：表示分配给应用线程的栈内存总量；
- summary.graphics：用于存储图形和窗口系统相关的内存，例如位图和GPU相关内存
- summary.total-pss：总PSS

上面简述了几个Native内存需要关心的字段，通过一定规则上报后，由服务端进行聚合&计算。
## 四、指标建设
- Native内存水位变化：每隔X分钟上报一次虚拟内存&实际使用的屋里内存，服务端聚合&计算数据，形成水位图。
- Native内存峰值：单位时间内内存峰值、应用使用期间内存峰值等
- Native内存波动率：每隔X分钟的数据，标准差计算得到波动数据。

除了指标数据上报，还需要其他信息帮助查找问题根源，接下来介绍两个开源框架，用于排查Native内存问题。
## 五、raphael原理解析

### 5.1 概述
MemoryLeakDetector 是西瓜视频基础技术团队开发的一款 native 内存泄漏监控工具。通过代理内存分配、释放的核心方法、栈回溯、缓存管理等方式对Native内存进行监控。

**git地址：** https://github.com/bytedance/memory-leak-detector

此项目在git上已经2年多没有新的提交，我这边fork下来，做了一些简单的修改。

### 5.2 核心流程

#### 5.2.1 核心方法代理
```cpp
   bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "malloc",
                          (void *) malloc_proxy, HookResultCallback,
                          NULL);

    bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "free",
                          (void *) free_proxy, HookResultCallback,
                          NULL);

    bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "calloc",
                          (void *) calloc_proxy, HookResultCallback,
                          NULL);

    bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "memalign",
                          (void *) memalign_proxy, HookResultCallback,
                          NULL);

    bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "realloc_proxy",
                          (void *) realloc_proxy, HookResultCallback,
                          NULL);

    bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "mmap",
                          (void *) mmap_proxy, HookResultCallback,
                          NULL);

    bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "mmap64",
                          (void *) mmap64_proxy, HookResultCallbacjiak,
                          NULL);

    bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "munmap",
                          (void *) munmap_proxy, HookResultCallback,
                          NULL);

    bytehook_hook_partial(allow_filter_for_hook_all, NULL, NULL, "pthread_exit",
                          (void *) pthread_exit_proxy, HookResultCallback,
                          NULL);
```

之前看了一些其他三方的内存监控的框架，大部分只对基础的内存方法做了监控，比如malloc/free/realloc等，仅这些方法是无法全面监控内存使用情况的。为确让监控更加全面，Android原生内存管理的监控需涵盖更多内存操作，特别是mmap和munmap。常规内存分配函数如malloc和free等，尽管常用，但在虚拟内存的总使用中只占小部分。

上述方法中，还有一个比较特殊的**pthread_exit**，这里是为了处理栈内存相关的内容。
栈分为信号栈、执行栈
- 信号栈：信号栈是为处理信号而专门保留的内存区域。它独立于常规的执行栈，以防常规栈在信号处理时已满或不可用。
- 执行栈：执行栈是线程进行正常操作时使用的栈，包括调用函数和存储局部变量。

线程的栈内存通常通过mmap()分配。当线程终止时，其栈内存的释放并非全都是通过显式调用munmap()实现的，详情如下
- 信号栈：使用 pthread_exit(void *return_value) 结束线程时，信号栈通常通过 munmap 被立即释放。
- 执行栈：
   - 当线程状态为 THREAD_DETACHED 并且通过 pthread_exit(void *return_value) 结束时，执行栈通过 _exit_with_stack_teardown(void* stack, size_t sz) 直接释放。
   - 如果线程被其他线程通过 int pthread_join(pthread_t t, void** return_value) 回收，执行栈释放在 pthread_internal_remove_and_free 中进行，最终也是通过 munmap 实现。

故如果需要监听栈内存的释放，还需要对pthread_exit进行Hook
```cpp
static void pthread_exit_proxy(void *value) {
    BYTEHOOK_STACK_SCOPE();
    pthread_attr_t attr;
    //获取当前线程的属性
    if (running && pthread_getattr_np(pthread_self(), &attr) == 0) {
        pthread_setspecific(guard, (void *) 1);
        //从自定义的内存缓存记录中删除掉栈内存
        cache->remove((uintptr_t) attr.stack_base);
        pthread_attr_destroy(&attr);
        pthread_setspecific(guard, (void *) 0);
        MB_NATIVE_MEM_MONITOR::Log::info("itkxl","pthread_exit_proxy");
    }
    BYTEHOOK_CALL_PREV(pthread_exit_proxy, value);
}

```
除了刚才聊到的栈内存释放问题，还有一个申请内存重入问题。
在调用malloc申请内存的时候，系统会根据申请的size决定最终走到brk还是mmap。如果申请内存过大，系统会直接走到mmap，上述代码中对两者都进行了hook，就可能产生一个重入的问题，造成统计过大。
```cpp

void *malloc_proxy(size_t size) {
    BYTEHOOK_STACK_SCOPE();
    if (running && size >= limit && !(uintptr_t) pthread_getspecific(guard)) {
        //通过设置线程局部存储 guard 来防止由于监控代码本身导致的递归调用，例如，在监控 malloc 时调用 malloc以及走到mmap造成重复统计。
        pthread_setspecific(guard, (void *) 1);
        void *address = BYTEHOOK_CALL_PREV(malloc_proxy, size);
        if (address != NULL) {
            insert_memory_backtrace(address, size);
        }
        pthread_setspecific(guard, (void *) 0);
        return address;
    }
    return BYTEHOOK_CALL_PREV(malloc_proxy, size);
}


static void *mmap_proxy(void *ptr, size_t size, int port, int flags, int fd, off_t offset) {
    BYTEHOOK_STACK_SCOPE();
    if (running && !(uintptr_t) pthread_getspecific(guard)) {
        pthread_setspecific(guard, (void *) 1);
        void *address = BYTEHOOK_CALL_PREV(mmap_proxy, ptr, size, port, flags, fd, offset);
        if (address != MAP_FAILED) {
            insert_memory_backtrace(address, size);
        }
        pthread_setspecific(guard, (void *) 0);
        return address;
    }
    return BYTEHOOK_CALL_PREV(mmap_proxy, ptr, size, port, flags, fd, offset);
}

```
上述代码除了解决重入问题之外，还通过限定size大小，来避免频繁处理。

#### 5.2.2 缓存管理
在代理函数中拿到申请的内存时，需要对其内存地址以、大小、申请堆栈进行记录，用于后续追溯问题。在此时，raphael并没有直接做堆栈解析与聚合，而是在最后打印结果的时候才做这一步。从代理方法中可以看到，内部使用了锁，在这样一个关键的函数上上锁，对程序的性能还是会产生一定的影响。

除了基础的通过代码判定内存出现问题外，预设的缓存大小被使用完成，也可以认定出现问题。

```cpp
void MemoryCache::insert(uintptr_t address, size_t size, Backtrace *backtrace) {
   //从缓存信息对象池中获取一个对象，用于记录申请的内存信息，包括地址、大小、堆栈等
    AllocNode *node = alloc_cache->apply();
    if (node == nullptr) {
        return;
    }

    node->addr = address;
    node->size = size;

    uint depth = backtrace->depth > 2 ? backtrace->depth - 2 : 1;
    memcpy(node->trace, backtrace->trace + 2, depth * sizeof(uintptr_t));
    node->trace[depth] = 0;
    //将内存地址做hash处理，用于临时存储
    uint16_t alloc_hash = (address >> ADDR_HASH_OFFSET) & 0xFFFF;
    //hash 链表法
    pthread_mutex_lock(&alloc_mutex);
    node->next =alloc_table[alloc_hash];
    alloc_table[alloc_hash] = node;
    pthread_mutex_unlock(&alloc_mutex);
}
```

#### 5.2.3 结果打印
当代码检测发现内存问题时候，需要手动停止，raphael会将记录的信息写到之前预设好的路径文件中。
```cpp
/**
 * FILE *output: 指向输出文件的指针
 * AllocNode *alloc_node: 指向内存分配节点的指针，包含分配的地址、大小以及调用栈跟踪信息
 * void **dl_cache: 动态链接库缓存，用于优化符号解析性能
 **/
void write_trace(FILE *outputNode *alloc_node, void **dl_cache) {
    //#define STACK_FORMAT_HEADER "\n0x%016lx, %u, 1\n"
    // 输出内存分配的地址和大小信息    
    fprintf(output, STACK_FORMAT_HEADER, alloc_node->addr, alloc_node->size);
    // 遍历alloc_node中的trace数组，该数组存储了调用栈
    for (int i = 0; i < alloc_node->trace[i] != 0; ++i) {
        uintptr_t pc = alloc_node->trace[i];
        xdl_info_t info;
        // 尝试解析程序计数器地址对应的符号信息
        if (0 == xdl_addr((void *) pc, &info, dl_cache) || (uintptr_t) info.dli_fbase > pc) {
            // 如果解析失败或函数基地址大于pc，输出未知符号的格式
            fprintf(output, STACK_FORMAT_UNKNOWN, pc);
        } else {
            // 解析成功，检查模块名称是否存在
            if (nullptr == info.dli_fname || '\0' == info.dli_fname[0]) {
                // 模块名称不存在，输出为匿名模块
                fprintf(output,
                        STACK_FORMAT_ANONYMOUS,
                        pc - (uintptr_t) info.dli_fbase,
                        info.dli_fbase);
            } else {
                // 模块名称存在，检查符号名称是否存在
                if (nullptr == info.dli_sname || '\0' == info.dli_sname[0]) {
                    // 符号名称不存在，仅输出文件名
                    fprintf(output, STACK_FORMAT_FILE, pc - (uintptr_t) info.dli_sname,
                            info.dli_sname);
                } else {
                    // 符号名称存在，尝试对C++符号进行解析（demangling）
                    int s;
                    const char *symbol = __cxxabiv1::__cxa_demangle(info.dli_sname, nullptr,
                                                                    nullptr, &s);
                    // 检查符号地址与pc的关系，以确定是否输出行信息
                    if (0 == (uintptr_t) info.dli_saddr || (uintptr_t) info.dli_saddr > pc) {
                        fprintf(
                                output,
                                STACK_FORMAT_FILE_NAME,
                                pc - (uintptr_t) info.dli_fbase,
                                info.dli_fname,
                                symbol == nullptr ? info.dli_sname : symbol);
                    } else {
                        fprintf(
                                output,
                                STACK_FORMAT_FILE_NAME_LINE,
                                pc - (uintptr_t) info.dli_fbase,
                                info.dli_fname,
                                symbol == nullptr ? info.dli_sname : symbol,
                                pc - (uintptr_t) info.dli_saddr
                        );
                    }
                    // 如果进行了符号解析，释放解析产生的内存
                    if (symbol != nullptr) {
                        free((void *) symbol);
                    }
                }
            }
        }

    }
}
```

整理流程：
- 写入头部信息，包含分配的地址和大小
- 遍历调用栈
- 符号解析
- 按格式输出

上传的日志文件，需要使用框架提供的python脚本进行解析
```bash
## analysis report
##   -r: report path
##   -o: output file name
##   -s: symbol file dir
python3 library/src/main/python/raphael.py -r report -o leak-doubts.txt -s ./symbol/
```

输出结果中可以看到单个so整体的虚拟内存占用。

#### 5.2.4 /proc/pid/maps文件辅助排查问题
```cpp
void Raphael::dump_system(JNIEnv *env) {
    char path[MAX_BUFFER_SIZE];
    if (snprintf(path, MAX_BUFFER_SIZE, "%s/maps", mSpace) >= MAX_BUFFER_SIZE) {
        return;
    }

    FILE *target = fopen(path, "w");
    if (target == nullptr) {
        LOGGER("dump maps failed, can't open %s/maps", mSpace);
        return;
    }
    //访问自己进程下的maps文件，写入到预设路径下
    FILE *source = fopen("/proc/self/maps", "re");
    if (source == nullptr) {
        LOGGER("dump maps failed, can't open /proc/self/maps");
        return;
    }

    while (fgets(path, MAX_BUFFER_SIZE, source) != nullptr) {
        fprintf(target, "%s", path);
    }

    fclose(source);
    fclose(target);
}
```

也提供了python脚本对map信息进行解析
```bash
## analysis maps
##   -m: maps file path
python3 library/src/main/python/mmap.py -m maps
```
### 5.3 拓展
除了替换xHook解决增量Hook的问题，还做了如下修改
- 去除32位设备检测：目前32位设备占比极低，内存泄漏又具有一定通用性，为了减少32设备手机性能影响以及降低堆栈操作的复杂度，故取消了32位设备的检测，仅在64位设备上采样运行。
- 将上传/proc/self/maps改为smaps文件：smaps比maps有更加详细的内存数据，smaps的解析脚本在 [内存映射文件概述](/Memory/memory-basic-knowledge/内存映射文件概述) 已经放出，感兴趣可以看一下。这里有一个问题，smaps文件大小远远大于maps文件，上传会带来一定的流量影响，但是由于此类文件内部结构有极大的相似性，这样在压缩之后，可以带来极其可观的压缩率，可将其压缩后上传。
- 原raphael的日志需要上传后进行解析，由于目前服务端暂时无人帮忙支持，故写了一份端上解析的代码,可在上传日志之前将其基本信息进行输出。
```cpp
// description:
// Created by itkxl on 2023/11/28.
//
#include <iostream>
#include <fstream>
#include <vector>
#include <string>
#include <regex>
#include <set>
#include "parser.h"
#include "../log.h"
#include "stdio.h"

struct Frame {
    std::string address;
    std::string symbol; // 可能包括路径和函数名
};

struct Trace {
    std::string address;
    int size;
    int count;
    std::vector<Frame> frames;
};

std::set<std::string> system_group = {
        "libhwui.so",
        "libsqlite.so",
        "libstagefright.so",
        "libcamera_client.so",
        "libandroid_runtime.so"
};


std::string group_record(const Trace& trace) {
    std::string default_group = "extras";
    std::regex path_regex(R"(.+\/(.+\.(so|apk|oat)))", std::regex_constants::icase);
    std::smatch match;

    for (const auto& frame : trace.frames) {
        if (std::regex_search(frame.symbol, match, path_regex)) {
            std::string lib_name = match[1];
            if (lib_name == "libnative-mem-monitor.so") {
                continue;
            }
            if (frame.symbol.find("/data/") != std::string::npos) {
                return lib_name;
            }
            if (system_group.find(lib_name) != system_group.end()) {
                default_group = lib_name;
            }
        }
    }

    return default_group;
}


std::map<std::string, long> parser(const char *path){

    char filepath[1024];
    sprintf(filepath, "%s/report", path);
    std::ifstream file(filepath);

    std::map<std::string, long> traceGroups;

    if (!file.is_open()){
        std::cerr << "无法打开文件" << std::endl;
        return traceGroups;
    }

    std::vector<Trace> traces;
    std::regex header_regex(R"(^(0x[0-9a-f]+),\s*(\d+),\s*(\d+)$)", std::regex_constants::icase);
    std::regex stack_regex(R"(^(0x[0-9a-f]+)\s+(.+)$)", std::regex_constants::icase);
    std::string line;


    Trace currentTrace;
    bool isNewTrace = true;

    while (std::getline(file, line)) {
        std::smatch match;
        if (std::regex_search(line, match, header_regex)) {
            if (!isNewTrace) {
                traces.push_back(currentTrace);
                currentTrace.frames.clear();
            }
            currentTrace.address = match[1];
            currentTrace.size = std::stoi(match[2]);
            currentTrace.count = std::stoi(match[3]);
            isNewTrace = false;
        }
        else if (std::regex_match(line, match, stack_regex)) {
            Frame frame{ match[1], match[2] };
            currentTrace.frames.push_back(frame);
        }
    }
    if (!isNewTrace) {
        traces.push_back(currentTrace);
    }

    file.close();

    for (const auto& trace : traces) {
        std::string group_key = group_record(trace);
        traceGroups[group_key] += trace.size;
    }

    return traceGroups;
}

```