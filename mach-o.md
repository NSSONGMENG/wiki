
![mach-o.jpg](https://github.com/NSSONGMENG/wiki/blob/master/images/mach-o.jpg)

* Mach-O 头（Mach Header）：这里描述了 Mach-O 的 CPU 架构、文件类型以及大小端等信息
* 加载命令（Load Command）：描述了如何加载每个段的信息
* 数据区（Segment和section）：包含每个段自身的信息，包括数据、代码以及段的执行权限等

**不仅仅是可执行文件是Macho-O，目标文件(.o)以及动态库，静态库都是Mach-O格式。**


## Header

```C
#define MH_MAGIC    0xfeedface    // the mach magic number
#define MH_CIGAM    0xcefaedfe    // NXSwapInt(MH_MAGIC)

// 代表mach-o文件的元信息，作用是让内核在读取该文件创建虚拟进程空间时，检查文件的合法性，以及当前硬件特性是否支持程序的运行
struct mach_header_64 {
    uint32_t    magic;        // 大端序(0xfeedfacf(MH_MAGIC_64))还是小端序(0xcffaedfe(MH_CIGAM_64))，iOS上是小端序
    cpu_type_t  cputype;      // CPU 类型标识符，同通用二进制格式中的定义
    cpu_subtype_t    cpusubtype;    // CPU 子类型标识符，ARM、x86、i386等宏定义划分
    uint32_t    filetype;    // 文件类型
    uint32_t    ncmds;       // LoadCommands的条数，每个LoadCommand代表一种Segment的加载方式
    uint32_t    sizeofcmds;  // LoadCommand的总大小，主要用于划分Mach-o文件的区域
    uint32_t    flags;       // 用于标记dyld加载过程
    uint32_t    reserved;    // 64 位的保留字段
};

 Mach-O 支持多种类型文件，所以此处引入了 filetype 字段来标明，定义如下：
#define    MH_OBJECT    0x1    // Target 文件：编译器对源码编译后得到的中间结果
#define    MH_EXECUTE   0x2    // 可执行二进制文件
#define    MH_FVMLIB    0x3    // VM 共享库文件（还不清楚是什么东西）
#define    MH_CORE      0x4    // Core 文件，一般在 App Crash 产生
#define    MH_PRELOAD   0x5    // preloaded executable file
#define    MH_DYLIB     0x6    // 动态库
#define    MH_DYLINKE   0x7    // 动态连接器 /usr/lib/dyld
#define    MH_BUNDLE    0x8    // 非独立的二进制文件，往往通过 gcc-bundle 生成
#define    MH_DYLIB_STUB    0x9    // 静态链接文件（还不清楚是什么东西）
#define    MH_DSYM      0xa        // 符号文件以及调试信息，在解析堆栈符号中常用
#define    MH_KEXT_BUNDLE   0xb    // x86_64 内核扩展

 flags 中所取值的全部定义
#define    MH_NOUNDEFS        0x1        // Target 文件中没有带未定义的符号，常为静态二进制文件
#define    MH_SPLIT_SEGS      0x20       // Target 文件中的只读 Segment 和可读写 Segment 分开  
#define    MH_TWOLEVEL        0x80       // 该 Image 使用二级命名空间(two name space binding)绑定方案
#define    MH_FORCE_FLAT      0x100      // 使用扁平命名空间(flat name space binding)绑定（与 MH_TWOLEVEL 互斥）
#define    MH_WEAK_DEFINES    0x8000     // 二进制文件使用了弱符号
#define    MH_BINDS_TO_WEAK   0x10000    // 二进制文件链接了弱符号
#define    MH_ALLOW_STACK_EXECUTION    0x20000    // 允许 Stack 可执行
#define    MH_PIE                      0x200000   // 对可执行的文件类型启用地址空间 layout 随机化
#define    MH_NO_HEAP_EXECUTION        0x1000000  // 将 Heap 标记为不可执行，可防止 heap spray 攻击
```


## Data区
```C
#define    SEG_PAGEZERO    "__PAGEZERO"     // 当是 MH_EXECUTE 文件时，捕获到空指针
#define    SEG_TEXT        "__TEXT"         // 代码/只读数据段
#define    SEG_DATA        "__DATA"         // 数据段
#define    SEG_OBJC        "__OBJC"         // Objective-C runtime用到的一些数据
#define    SEG_LINKEDIT    "__LINKEDIT"     // 包含需要被动态链接器使用的符号表、字符串表、重定位表等

struct segment_command_64 {
    uint32_t    cmd;        // LC_SEGMENT_64
    uint32_t    cmdsize;    // section_64 结构体所需要的空间
    char        segname[16];    // segment 名字，上述宏中的定义
    uint64_t    vmaddr;    // 所描述段的虚拟内存地址
    uint64_t    vmsize;    // 为当前段分配的虚拟内存大小
    uint64_t    fileoff;   // 当前段在文件中的偏移量
    uint64_t    filesize;  // 当前段在文件中占用的字节     
    vm_prot_t   maxprot;   // 段所在页所需要的最高内存保护，用八进制表示
    vm_prot_t   initprot;  // 段所在页原始内存保护
    uint32_t    nsects;    // 段中 Section 数量
    uint32_t    flags;     // 标识符
};

struct section_64 {
    char        sectname[16];    // Section 名字
    char        segname[16];     // Section 所在的 Segment 名称
    uint64_t    addr;            // Section 所在的内存地址
    uint64_t    size;            // Section 的大小
    uint32_t    offset;          // Section 所在的文件偏移
    uint32_t    align;           // Section 的内存对齐边界 (2 的次幂)
    uint32_t    reloff;          // 重定位信息的文件偏移
    uint32_t    nreloc;          // 重定位条目的数目
    uint32_t    flags;           // 标志属性
    uint32_t    reserved1;       // 保留字段1 (for offset or index)
    uint32_t    reserved2;       // 保留字段2 (for count or sizeof)
    uint32_t    reserved3;       // 保留字段3
};
```

## Segment & section
Segment Type | description
---- | ---- |
`__TEXT.__text` | 程序代码
`__TEXT.__cstring` | c字符串
`__DATA.__cfstring` | 程序中使用的 Core Foundation 字符串（CFStringRefs）
`__TEXT.__const` | const关键字修饰的常量
`__TEXT.__unwind_info` | 存储异常处理信息
`__TEXT.__eh_frame` | 调试辅助信息
`__TEXT.__objc_methname` | 方法名称
`__TEXT.__objc_methtype` | O方法类型
`__TEXT.__objc_classname` | 类名称
`__DATA.__data` | 初始化过的可变数据
`__DATA.__la_symbol_ptr` | lazy binding的指针表，表中的指针一开始都指向__stub_helper
`__DATA.__nl_symbol_ptr` | 非lazy binding的指针表，每个表项中的指针都指向一个在装载过程中被动态链机器搜索完成的符号
`__DATA.__const` | 存放一些需要在类加载过程中用到的只读数据
`__DATA.__bss` | BSS，存放为初始化的全局变量，即常说的静态内存分配
`__DATA.__common` | 没有初始化过的符号声明
`__DATA.__objc_imginfo` | 镜像信息
`__DATA.__objc_classlist` | 全部类列表（metaclass也是一种class）, 对应gdb_objc_realized_classes数组
`__DATA.__objc_classrefs` | 被引用的类列表， 对应noClassesRemapped
`__DATA.__objc_catlist` | 分类列表
`__DATA.__objc_protolist` | 协议列表
`__DATA.__objc_superrefs` | 超类引用，维护父子类关系的信息
`__DATA.__objc_selrefs` | 标记SEL对应的字符串被引用
`__DATA.__objc_protorefs` | 协议引用
`__TEXT.__stubs` | 用于 Stub 的占位代码，很多地方称之为桩代码。本质上是一小段会直接跳入lazybinding的表对应项指针指向的地址的代码。
`__TEXT.__stubs_helper` | 当 Stub 无法找到真正的符号地址后的最终指向
