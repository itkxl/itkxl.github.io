## 一、背景
随着业务场景逐渐复杂，开发团队越来越依赖于原生代码（如C和C++）来提高性能和优化资源使用。跨端&业务&三方等场景Native代码越来越多，建设Native层的内存监控非常重要。
## 二、目标
- 提供Native内存使用数据：让开发者能够实时了解应用的内存使用情况，包括总内存使用量、峰值使用量等。
- 能及时发现线上出现的Native内存问题：内存泄漏&内存不合理使用
## 三、数据采集
这部分可以先看一下之前写的[内存映射文件概述](../memory-mapping-file/内存映射文件概述.md)

- 命令行获取
```bash
adb shell dumpsys meminfo pid
adb shell dumpsys meminfo packageName

//以com.android.settings为例
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


## 四、指标建设
## 五、raphael源码解析
## 六、KOOM Native源码解析