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
    if ('[' == path_name[0] || 0 == path_name_len) {
        continue;
    }

    //6、如果so库名称&方法匹配成功且完成基地址查找，进入hook流程
    if (!regexec(&path_name_regex, path_name, 0, NULL, 0)) {
        do_hook(base_address, symbol, new_function);
    }
}
}
```

### 2.2 查找.dynamic段
在上一小节中，已经找到了待处理的so库路径以及其基地址，本节将根据so库的路径以及基地址，查找so文件的.dynamic段，用于进一步解析外部调用的信息。

.dynamic段包含了许多重要信息，例如重定位表（.rela.plt）、符号表（.dynsym）、字符串表（.dynstr）等的地址和大小。
```cpp
void do_hook(uintptr_t base_address,const char* symbol, void* new_function){
    //1、根据ELF Hader 查找ELF Program Header Table,后者是前者的基地址+偏移量
    ElfW(Ehdr) *elf_header = base_address;
    ElfW(Phdr) *elf_program_header_table= base_address + elf_header->e_phoff;
    int program_header_table_length = elf_header->e_phnum;


    uintptr_t dyn_address;
    unsigned int dyn_table_length;

    //2、遍历Program Header Table,找到其中的dynmic类型的段
    for (int i = 0; i < program_header_table_length; ++i) {
        if (elf_program_header_table[i].p_type == PT_DYNAMIC) {
            dyn_address = elf_program_header_table[i].p_vaddr + base_address;
            dyn_table_length = elf_program_header_table[i].p_memsz / sizeof(ElfW(Dyn));
            break;
        }
    }
    //... 查找.rela.plt
}

```
### 2.3 查找.rela.plt
在2.2中，找到了.dynamic段，此节需要从.dynmiac段中查找到.rela.plt段。

.rela.plt 是一个关键的表，在 ELF 文件中用于管理动态链接时函数地址的更新。当我们的代码调用了外部函数的时候，.rela.plt 提供了必要的信息来更新 .got.plt 表，使得程序能够正确地跳转到这些函数的实际地址。

```cpp
    //
    ElfW(Dyn) *dyn_table = dyn_address;
    ElfW(Dyn) *dyn_table_end = dyn_table + dyn_table_length;
    uintptr_t    rel_table_address;
    uintptr_t    sys_table_address;
    uintptr_t    str_table_address;
    unsigned int rel_table_length;
    for (; dyn_table < dyn_table_end; dyn_table++) {
        switch (dyn_table->d_tag) {
            // 此时已经读取到结尾
            case DT_NULL: 
                dyn_table_end = dyn_table;
                break;
            //指向 .rela.plt 表的地址。该地址是相对于基地址的偏移量，需要加上基地址 base_address 转换为绝对地址。
            case DT_JMPREL: 
                rel_table_address = dyn_table->d_un.d_ptr + base_address;
                break;
            //DT_PLTRELSZ: 存储 .rela.plt 表的字节大小，通过除以 sizeof(ElfW(Rela)) 转换为表中的条目数。
            case DT_PLTRELSZ: // .rela.plt size 但不是 length
                rel_table_length = dyn_table->d_un.d_val / sizeof(ElfW(Rela));
                break;
            //DT_SYMTAB: 指向符号表（.dynsym）的地址，处理方式同 DT_JMPREL。
            case DT_SYMTAB:
                sys_table_address = dyn_table->d_un.d_ptr + base_address;
                break;
            //DT_STRTAB: 指向字符串表（.dynstr）的地址，处理方式同 DT_JMPREL。
            case DT_STRTAB:
                str_table_address = dyn_table->d_un.d_ptr + base_address;
                break;
        }
    }
```
### 2.4 处理目标函数
每个 .rela.plt 条目通过 r_info 字段关联到 .dynsym（动态符号表）中的一个条目。
根据其中信息解析.dynsym & .dynstr，找到目标函数的信息，对其调用地址进行替换。

```cpp
    ElfW(Rela) *rel_table = rel_table_address;

    //1、遍历 .rela.plt 表
    for (int i = 0; i < rel_table_length; ++i) {
        //2、获取符号索引和符号表项
        int sys_table_index = ELF_R_SYM(rel_table[i].r_info);
        ElfW(Sym) *sys_item = sys_table_address + sys_table_index * sizeof(ElfW(Sym));

        //3、拼接函数名
        char* fun_name = sys_item->st_name + str_table_address;

        //4、匹配函数名，如果匹配成功，修改函数地址
        if (memcmp(symbol, fun_name, strlen(symbol)) == 0) {
            //计算需要修改的 .got.plt 条目的内存地址。
            uintptr_t mem_page_start = rel_table[i].r_offset + base_address;
            //修改内存页的保护权限，以允许写操作。
            mprotect(PAGE_START(mem_page_start), getpagesize(), PROT_READ | PROT_WRITE);
            //将 .got.plt 条目更新为 new_function 的地址。
            *(void **)(rel_table[i].r_offset + base_address) = new_function;
            //清除 CPU 指令缓存，确保刚才的修改立即生效。
            __builtin___clear_cache(PAGE_START(mem_page_start), PAGE_START(mem_page_start) + getpagesize());
            break;
        }
    }
```


## 三、xHook源码分析
xHook 是爱奇艺开源的一个针对 Android 平台 ELF (可执行文件和动态库) 的 PLT (Procedure Linkage Table) hook 库。
内部做了大量稳定性处理，可以用于线上。
接下来对其核心方法进行简单的解析，可以看到其流程与上面写的简易代码基本一致。

### 3.1 核心代码解析

对于解析map文件，查到dynamic、rel.plt信息等代码省略

```cpp
/**
 * 查找目标函数
 * params:
 *      xh_elf_t *self: 表示目标 ELF 文件的上下文或状态，包含关于该文件的各种信息
 *      const char *symbol: 需要替换的函数名。
 *      void *new_func: 新函数的地址。
 *      void **old_func: 用于存储被替换函数原地址的指针。
 */
int xh_elf_hook(xh_elf_t *self, const char *symbol, void *new_func, void **old_func)
{
    uint32_t                        symidx;
    void                           *rel_common;
    xh_elf_plain_reloc_iterator_t   plain_iter;
    xh_elf_packed_reloc_iterator_t  packed_iter;
    int                             found;
    int                             r;

    //省略了部分基础判断代码
    
    //通过符号名查找对应的符号索引。如果查找失败，函数返回。
    if(0 != (r = xh_elf_find_symidx_by_name(self, symbol, &symidx))) return 0;
    
    //遍历 .rel.plt 和 .rel.dyn 重定位表
    if(0 != self->relplt)
    {
        xh_elf_plain_reloc_iterator_init(&plain_iter, self->relplt, self->relplt_sz, self->is_use_rela);
        //对于 .rel.plt 和 .rel.dyn 重定位表，使用 xh_elf_plain_reloc_iterator_init 函数初始化一个迭代器，然后逐个处理每一个重定位条目：
        while(NULL != (rel_common = xh_elf_plain_reloc_iterator_next(&plain_iter)))
        {
            if(0 != (r = xh_elf_find_and_replace_func(self,
                                                      (self->is_use_rela ? ".rela.plt" : ".rel.plt"), 1,
                                                      symbol, new_func, old_func,
                                                      symidx, rel_common, &found))) return r;
            if(found) break;
        }
    }

    //省略.rel(a).dyn、.rel(a).android的处理，整体与.rel.plt类似
    return 0;
}


/**
 * 执行方法替换
 * xh_elf_t *self: 指向当前 ELF 处理结构的指针，包含了关于目标 ELF 文件的状态和属性。
 * const char *section: 被操作的重定位表的名称（如 .rel.plt 或 .rel.dyn）。
 * const char *symbol: 需要替换的符号名称。
 * void *new_func: 新函数的地址。
 * void **old_func: 指向存储原始函数地址的指针的位置。
 * uint32_t symidx: 符号在符号表中的索引。
 * void *rel_common: 指向当前处理的重定位条目。
 * int *found: 输出参数，用于指示是否找到了指定的符号。
 */
static int xh_elf_find_and_replace_func(xh_elf_t *self, const char *section,
                                        int is_plt, const char *symbol,
                                        void *new_func, void **old_func,
                                        uint32_t symidx, void *rel_common,
                                        int *found)
{
    ElfW(Rela)    *rela;
    ElfW(Rel)     *rel;
    ElfW(Addr)     r_offset;
    size_t         r_info;
    size_t         r_sym;
    size_t         r_type;
    ElfW(Addr)     addr;
    int            r;

    if(NULL != found) *found = 0;
    
    //根据 self->is_use_rela 判断使用的是哪种类型的重定位条目，REL 还是 RELA。RELA 类型包含额外的加数（r_addend），而 REL 不包含。
    if(self->is_use_rela)
    {
        rela = (ElfW(Rela) *)rel_common;
        //提取重定位条目的信息，包括 r_info（包含符号索引和重定位类型）和 r_offset（需要修正的地址偏移）。
        r_info = rela->r_info;
        r_offset = rela->r_offset;
    }
    else
    {
        rel = (ElfW(Rel) *)rel_common;
        r_info = rel->r_info;
        r_offset = rel->r_offset;
    }

    //从 r_info 中提取符号索引（r_sym），如果该索引与提供的 symidx 不匹配，则直接返回，表示未找到对应符号。
    r_sym = XH_ELF_R_SYM(r_info);
    if(r_sym != symidx) return 0;

    //从 r_info 中提取重定位类型（r_type）。根据 is_plt 参数判断应该匹配的重定位类型，确保条目类型符合期望（例如，对 PLT 条目应该是 JUMP_SLOT）。
    r_type = XH_ELF_R_TYPE(r_info);
    if(is_plt && r_type != XH_ELF_R_GENERIC_JUMP_SLOT) return 0;
    if(!is_plt && (r_type != XH_ELF_R_GENERIC_GLOB_DAT && r_type != XH_ELF_R_GENERIC_ABS)) return 0;

    //we found it
    XH_LOG_INFO("found %s at %s offset: %p\n", symbol, section, (void *)r_offset);
    if(NULL != found) *found = 1;

    //执行替换操作
    addr = self->bias_addr + r_offset;
    //计算需要修改的实际内存地址（self->bias_addr + r_offset），并检查地址是否有效。
    if(addr < self->base_addr) return XH_ERRNO_FORMAT;
    //调用 xh_elf_replace_function 执行实际的替换操作，如果替换失败则记录错误并返回相应的错误码。
    if(0 != (r = xh_elf_replace_function(self, symbol, addr, new_func, old_func)))
    {
        XH_LOG_ERROR("replace function failed: %s at %s\n", symbol, section);
        return r;
    }

    return 0;
}


/**
 * 最终执行替换的方法
 * xh_elf_t *self: 指向当前 ELF 文件处理结构的指针，包含了 ELF 文件的相关信息。
 * const char *symbol: 正在被替换的符号名称，用于日志记录。
 * ElfW(Addr) addr: 需要修改的内存地址，该地址指向旧函数。
 * void *new_func: 新函数的地址，即将被写入到 addr 指向的位置。
 * void **old_func: 指针的指针，用于存储被替换的旧函数地址。
 */
static int xh_elf_replace_function(xh_elf_t *self, const char *symbol, ElfW(Addr) addr, void *new_func, void **old_func)
{
    void         *old_addr;
    unsigned int  old_prot = 0;
    unsigned int  need_prot = PROT_READ | PROT_WRITE;
    int           r;

    //首先检查在 addr 指向的地址处存储的内容是否已经是 new_func。如果是，return
    if(*(void **)addr == new_func) return 0;

    //使用 xh_util_get_addr_protect 函数获取当前地址的内存保护权限，并存储在 old_prot 中。
    if(0 != (r = xh_util_get_addr_protect(addr, self->pathname, &old_prot)))
    {
        XH_LOG_ERROR("get addr prot failed. ret: %d", r);
        return r;
    }
    
    //如果当前的内存保护权限不包括读写（PROT_READ | PROT_WRITE），则尝试设置新的权限以允许对该内存地址的写操作。
    if(old_prot != need_prot)
    {
        //set new prot
        if(0 != (r = xh_util_set_addr_protect(addr, need_prot)))
        {
            XH_LOG_ERROR("set addr prot failed. ret: %d", r);
            return r;
        }
    }
    
    //在修改内存权限后，保存 addr 处当前的函数地址（old_addr），如果提供了 old_func 参数，则通过该参数返回。
    old_addr = *(void **)addr;
    if(NULL != old_func) *old_func = old_addr;

    // ！！！！然后将 new_func 写入 addr 指向的内存位置。！！！！ 
    *(void **)addr = new_func; //segmentation fault sometimes 可能发生段错误

    //替换函数地址后，尝试将内存保护权限恢复为原来的 old_prot。
    if(old_prot != need_prot)
    {
        //restore the old prot
        if(0 != (r = xh_util_set_addr_protect(addr, old_prot)))
        {
            XH_LOG_WARN("restore addr prot failed. ret: %d", r);
        }
    }
    
    //由于修改了执行代码的内存，为了确保处理器的指令缓存中不会残留旧的指令，调用 xh_util_flush_instruction_cache 刷新相关的缓存
    xh_util_flush_instruction_cache(addr);

    XH_LOG_INFO("XH_HK_OK %p: %p -> %p %s %s\n", (void *)addr, old_addr, new_func, symbol, self->pathname);
    return 0;
}
```

### 3.2 bhook
ByteHook是xHook的升级版本，解决xHook的一些痛点问题：
- xHook对单个函数多次Hook引发的冲突问题：App中如果多个场景Hook了同一个方法，其中一处进行unHook操作，会影响其他地方的行为；
- xHook无法自动Hook后置加载的so：需要手动调用refresh；
- 代码问题可能造成代理函数之间的递归调用和环形调用；

git地址：https://github.com/bytedance/bhook

目前手上项目基本已经完全改为bhook.