## 一、概述
这是线下使用eBPF工具opensnoop查看App文件使用情况，环境配置和流程走通踩了一些坑，在次记录。
## 二、设备要求
- 设备：Pixel 6 （Root）
- 内核：5.10.157
  
## 三、环境搭建&运行

### 3.1 ExtendedAndroidTools
[BCC git地址](https://github.com/iovisor/bcc)  

截至目前，bcc官网依旧不支持Android交叉编译。

需要使用到三方的仓库 [ExtendedAndroidTools](https://github.com/facebookexperimental/ExtendedAndroidTools)  

用于在Android上运行bcc和bpftools 可以在其git地址中的aciton下，找到已经编译好的，选择对应的cpu架构即可。

下载完成后,将压缩包推进手机并解压

```bash
adb push boftools-arm.tar.gz /data/local/tmp
adb shell
cd /data/local/tmp
tar xf bpftools-atm64.tar.gz
```
此时，解压出来的内容已经包含ebpf相关工具。

### 3.2 运行opensnoop

```bash
adb shell
cd /data/local/tmp/bpftools  
//bcc相关的工具都放在 bpftools/share/bcc/tools下  
//这里可以在btpftools目录下，尝试运行  
.python3 share/bcc/tools/opensnoop
```

此时报错了，坑点也在这里。

报错内容如下：
```bash
cannot attach kprobe, probe entry may not exist
Traceback (most recent call last):
  File "/data/local/tmp/bpftools/share/bcc/tools/opensnoop", line 389, in <module>
    b.attach_kprobe(event=fnname_open, fn_name="syscall__trace_entry_open")
  File "/data/local/tmp/bpftools/lib/python3.10/site-packages/bcc/__init__.py", line 845, in attach_kprobe
    raise Exception("Failed to attach BPF program %s to kprobe %s"
Exception: Failed to attach BPF program b'syscall__trace_entry_open' to kprobe b'sys_open', it's not traceable (either non-existing, inlined, or marked as "notrace")
```

无法将kprobe的 ‘sys_open’和挂载点 'syscall_trace_entry_open'链接起来
主要原因是在挂载的的时候，计算'sys_open'方法名错误
报错是在 opensnoop的389行，此处我们使用  
cat share/bcc/tools/opensnoop 查看一下源码  
```python
b.attach_kprobe(event=fnname_open, fn_name="syscall__trace_entry_open")
```
fnname_open 和syscall__trace_entry_open，在attach_kprobe的时候失败了
失败的原因是 fnname_open获取的值不对，获取的名字错误。


在opensnoop源码的291行
```python
fnname_open = b.get_syscall_prefix().decode() + 'open'
```
这种形式在Android上运行会出现问题(x86上没问题)

下面我们可以在x86上验证一个 get——syscall_prefix().decode()值的情况
找一台x86的ubuntu设备

```shell
sudo python3
from bcc import BPF
b = BPF(text ='')
b.get_syscall_prefix().decode()
```
获得到的值为'__x64_sys_'

可以看到其前面为  __x64_  ，Android中少了这部分  
核心原因是get_syscall_prefix在Android上执行出现了问题  

此处看一下bcc这部分的源码 [https://github.com/iovisor/bcc/blob/master/src/python/bcc/__init__.py](https://github.com/iovisor/bcc/blob/master/src/python/bcc/__init__.py)


```python

 _syscall_prefixes = [
        b"sys_",
        b"__x64_sys_",
        b"__x32_compat_sys_",
        b"__ia32_compat_sys_",
        b"__arm64_sys_",
        b"__s390x_sys_",
        b"__s390_sys_",
    ]

    ...

def get_syscall_prefix(self):
        for prefix in self._syscall_prefixes:
            if self.ksymname(b"%sbpf" % prefix) != -1:
                return prefix
        return self._syscall_prefixes[0]


```
词语应该是在执行ksynmname的时候，没有读取到内核的符号信息(/proc/kallsyms)，导致返回了-1
之后直接返回了数组中第0个元素， 及  sys_  
与上面的错误关联起来，也就是sys_open找不到。


**解决办法：**

cat /proc/sys/kernal/kptr_restrict  
在x86上，返回的1
在Android上，默认返回了2

0 ： 普通用户和root用户都可以通过程序来读取  
1 ： 仅root用户可以通过程序读取  
2 ： 都不可以

此处将其为改为0即可

```bash
echo 0 > /proc/sys/kernel/kptr_restrict
```

之后再运行.python3 share/bcc/tools/opensnoop即可。



### 3.3 最终效果

```bash
22897  KH_IDyResMgr      493   0 /data/user/0/com.duowan.kiwi/app_dynamic_so_64/libHYNN_SR_SNPE.so
22897  KH_IDyResMgr      493   0 /data/user/0/com.duowan.kiwi/app_dynamic_so_64/libHYNN_QNN.so
22897  KH_IDyResMgr      493   0 /data/user/0/com.duowan.kiwi/app_dynamic_so_64/libcpucl.so
22897  KH_IDyResMgr      493   0 /data/user/0/com.duowan.kiwi/app_dynamic_so_64/libcpucl.so
22897  com.duowan.kiwi    -1  13 /system/build.prop
714    NodeLooperThrea     9   0 /sys/devices/platform/17000010.devfreq_mif/devfreq/17000010.devfreq_mif/min_freq
23651  SharedPreferenc    66   0 /data/user/0/com.duowan.kiwi/shared_prefs/logNewModel.xml
22897  KH_IDyResMgr      493   0 /data/user/0/com.duowan.kiwi/app_dynamic_so_64/libhcl.so
22897  KH_IDyResMgr      493   0 /data/user/0/com.duowan.kiwi/app_dynamic_so_64/libhcl.so
23651  .kiwi:remoteweb    66   0 /storage/emulated/0
23651  .kiwi:remoteweb    66   0 /storage/emulated/0
23651  .kiwi:remoteweb    68   0 /storage/emulated/0
23651  .kiwi:remoteweb    68   0 /storage/emulated/0
23651  .kiwi:remoteweb    68   0 /sys/devices/system/cpu/possible
22897  KH_IDyResMgr      493   0 /data/user/0/com.duowan.kiwi/app_dynamic_so_64/libhuaweiNPU.so
22897  KH_IDyResMgr      493   0 /data/user/0/com.duowan.kiwi/app_dynamic_so_64/libhuaweiNPU.so
22897  KH_IDyResMgr      505   0 /data/user/0/com.duowan.kiwi/app_dynamic_assets
22897  KH_IDyResMgr      505   0 /data/user/0/com.duowan.kiwi/app_dynamic_assets/DyAssets_Enhance.zip
23651  .kiwi:remoteweb    72   0 /data/app/~~3P0f1Ht4TyBGqI7nEU41WQ==/com.duowan.kiwi-oEvYO8EGK3VZ_6h0T7i7UA==/lib/arm64/libc++_shared.so
23651  .kiwi:remoteweb    72   0 /data/app/~~3P0f1Ht4TyBGqI7nEU41WQ==/com.duowan.kiwi-oEvYO8EGK3VZ_6h0T7i7UA==/lib/arm64/libc++_shared.so
23651  .kiwi:remoteweb    72   0 /data/app/~~3P0f1Ht4TyBGqI7nEU41WQ==/com.duowan.kiwi-oEvYO8EGK3VZ_6h0T7i7UA==/lib/arm64/libmarsxlog.so
23651  .kiwi:remoteweb    72   0 /data/app/~~3P0f1Ht4TyBGqI7nEU41WQ==/com.duowan.kiwi-oEvYO8EGK3VZ_6h0T7i7UA==/lib/arm64/libmarsxlog.so
```

手机中，目前打开的所有的文件都可以被监听到，如果想过滤进程可以使用
```
.python3 share/bcc/tools/opensnoop -p 进程id
```