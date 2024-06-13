## 一、概述
在之前做的IO监控，Dump Hprof文件并裁剪等内容都使用到了PLT Hook，其在多项APM能力建设中提供了非常大的帮助，为了加深对其的理解，文本将按照其原理实现一下其核心逻辑。

PLT Hook主要应用于跨so调用，或者叫做外部调用，比如liba.so调用了libc.so中的文件打开open方法，如果是so内部调用，则使用PLT HOOK是无能无力的。

在之前介绍GOT/PLT表的时候，有概述过外部调用的符号以及地址会写到.got或者是.got.plt表中，故实现PLT 的核心原理就是将.got.plt表中的函数地址替换掉，达到Hook的目的。

## 二、基本流程
按照概述中概述的原理，我们需要从运行中的内存中，找到并替换相关函数在.got或者是.got.plt中的外部调用的函数地址，整体上需要按照如下流程。
- 1、查找共享库在内存中的基地址；
- 2、根据ELF文件头信息，查找类型为PT_DYNAMIC的dynamic段
- 3、解析dynamic段内容，找到.rela.plt、dynsym等段
- 4、遍历 .rela.plt 表，查找需要被Hook的函数信息
- 5、修改.got.plt内容，达到Hook目的

### 2.1 查找基地址
在之前介绍内存监控的时候，有一篇内存映射文件的概述，里面有介绍一个非常重要的文件/proc/self/maps,其内部包含了共享库加载的内存信息。其单条格式如下
```shell
7718e00000-7718f6a000 r--p 00000000 fe:22 84   /apex/com.android.art/lib64/libart.so
```
- 7718e00000-7718f6a000:起始地址 前后值相减可获得大小
- r--p:权限
- 00000000:偏移量
- fe:22:内存区域所在的设备编号，通常是磁盘上的设备。
- 84:节点，文件系统中的innode号，指向具体的文件或者其他文件系统
- libart.so: 关联的文件路径

我们通过解析这个文件中的内容，即可获取到我们想Hook的so文件的基地址（起始地址）。
```cpp
void hook(){
    //1、打开proc/self/maps文件
    FILE* maps_file = fopen("/proc/self/maps", "r");

    char        line[512];
    uintptr_t   base_address;
    char        permission[5];
    int         path_name_position;

    //2、按行读取内容
    while (fgets(line, sizeof(line), maps_file)) {

    //3、按照前面介绍的格式，将每行内容进行格式化匹配，主要为了玻璃起始地址、偏移量、权限、文件名
    if(sscanf(line, "%"PRIxPTR"-%*lx %4s %lx %*x:%*x %*d%n", &base_address, permission, &offset, &path_name_position) != 3){
        continue;
    }

    //4、判断权限以及偏移量，偏移量需要选0的，非0的可能是其他场景mmap打开文件，非文件的基地址
    if (permission[0] != 'r' || permission[3] != 'p' || offset != 0)) {
            continue;
    }

    //5、获取文件名称，并处理一些异常情况
    while (isspace(line[path_name_position]) && path_name_position < sizeof(line) - 1){
            path_name_position += 1;
    }
    if (path_name_position >= sizeof(line) + 1) {
        continue;
    }
    path_name = line + path_name_position;
    path_name_len = strlen(path_name);
    if(path_name_len == 0) continue;
    if (path_name[path_name_len -1] == '\n') {
            path_name[path_name_len - 1] = '\0';
            path_name_len -= 1;
    }
        
        // 异常的这种情况处理 [anon:dalvik-main space (region space)]
        if ('[' == path_name[0] || 0 == path_name_len) {
            continue;
        }

        if (0 == regexec(&path_name_regex, path_name, 0, NULL, 0)) {
            core_dhook(base_address, symbol, new_function);
        }
}
}
```


### 2.2 查找.dynamic段
### 2.3 
### 2.4


## 三、xHook源码分析