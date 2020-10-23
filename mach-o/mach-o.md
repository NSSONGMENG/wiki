# 可执行文件：
在unix中，任何文件都可以通过简单的chmod+x命令标记为可执行文件。这个标志指示告诉系统内核可以将该文件读入内存，然后内核查找头签名，据此确定可执行文件的格式。头签名通常称为`魔数（magic）`，这个签名是预先定义好的常量值。文件被读入后，通过魔数帮助判断文件格式，如果是被支持的二进制格式，就会调用指定的加载器函数。

可执行格式 | 魔数 | 用途 |
---- | ---- | ---- |
脚本 | #！ | 主要用于脚本，如shell、php、python等，内核找到`#!`后面跟的字符串，并执行其表示的命令，文件剩下的部分通过标准输入(stdin)传递给这个命令 |
通用二进制格式（胖二进制格式，header后跟不同架构的二进制文件） | 0xcafebabe(小端序)、0xbebafeca(大端序) | OS X上支持的包含多种架构支持的二进制格式 |
Mach-o | 0xfeedface(32位)、0xfeedfacf(64位) | OS X、iOS原生二进制格式 |




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
    uint32_t    magic;        // 0xfeedfacf(MH_MAGIC_64)64位二进制；0xcffaedfe(MH_CIGAM_64)32位二进制
    cpu_type_t  cputype;      // CPU 类型，同通用二进制格式中的定义，定义于<mach/machine.h>中
    cpu_subtype_t    cpusubtype;    // CPU 子类型标识符，ARM、x86、i386等宏定义划分
    uint32_t    filetype;    // 文件类型（可执行文件、库文件、核心转储文件、内核扩展等）
    uint32_t    ncmds;       // LoadCommands的条数，每个LoadCommand代表一种Segment的加载方式
    uint32_t    sizeofcmds;  // LoadCommand的总大小，主要用于划分Mach-o文件的区域
    uint32_t    flags;       // 动态链接器dyld的标志
    uint32_t    reserved;    // 64 位的保留字段
};

> 当进程发生错误或收到“信号”(signal) 而终止执行时，系统会将核心映像写入一个文件，以作为调试之用，这就是所谓的核心转储(core dump)。

 Mach-O 支持多种类型文件，所以此处引入了 filetype 字段来标明，定义如下：
#define    MH_OBJECT    0x1    // 可重定位的目标文件：编译器对源码编译后得到的中间结果，也可以是32位的内核扩展
#define    MH_EXECUTE   0x2    // 可执行二进制文件
#define    MH_FVMLIB    0x3    // VM 共享库文件（还不清楚是什么东西）
#define    MH_CORE      0x4    // 核心转储文件，一般在核心转储是产生
#define    MH_PRELOAD   0x5    // preloaded executable file
#define    MH_DYLIB     0x6    // 动态库 /usr/lib中的库文件，以及框架中的二进制文件
#define    MH_DYLINKE   0x7    // 动态连接器 /usr/lib/dyld
#define    MH_BUNDLE    0x8    // 非独立的二进制文件，加载进其他二进制才能发挥作用，往往通过 gcc-bundle 生成
#define    MH_DYLIB_STUB    0x9    // 静态链接文件（还不清楚是什么东西）
#define    MH_DSYM      0xa        // 符号文件以及调试信息，在解析堆栈符号中常用
#define    MH_KEXT_BUNDLE   0xb    // 64位的内核扩展

 flags 中所取值的全部定义
#define    MH_NOUNDEFS        0x1        // 目标文件中没有带未定义的符号，常为静态二进制文件，没有进一步的链接依赖关系
#define    MH_SPLIT_SEGS      0x20       // 目标文件中的只读段和可读写段分开了
#define    MH_TWOLEVEL        0x80       // 该Image使用二级命名空间(two name space binding)绑定方案
#define    MH_FORCE_FLAT      0x100      // 使用扁平命名空间(flat name space binding)绑定（与 MH_TWOLEVEL 互斥）
#define    MH_WEAK_DEFINES    0x8000     // 二进制文件使用了弱符号
#define    MH_BINDS_TO_WEAK   0x10000    // 二进制文件链接了弱符号
#define    MH_ALLOW_STACK_EXECUTION    0x20000    // 允许栈可执行，只有可执行文件可以用这个标志
#define    MH_PIE                      0x200000   // 对可执行的文件类型启用地址空间布局随机化
#define    MH_NO_HEAP_EXECUTION        0x1000000  // 将堆标记为不可执行，可防止`堆喷heap spray` 攻击
```

## Load Command
![load_command_pic.png](https://github.com/NSSONGMENG/wiki/blob/master/images/load_command_pic.png)

\# | 命令 | 内核处理函数（定义在bsd/kern/mach_loader.c中） | 用途
---- | ---- | ---- | ---- |
0x01 | LC_SEGMENT | laod_segment | （处理32位的段）指导内核如何设置新进程的内存空间，这些段直接从Mach-o二进制文件加载到内存中 |
0x19 | LC_SEGMENT_64 | load_segment | 处理64位的段 |
0x0E | LC_LOAD_DYLINKER | load_dylinker | 调用dyld (usr/lib/dyld) |
0x1B | LC_UUID | 内核将UUID复制到内部表示mach目标的数据中 | 一个唯一的128位id，这个id匹配一个二进制文件及其对应的符号 |
0x04 | LC_THREAD | load_thread | 开启一个Mach线程，但是不分配栈（很少在核心转储之外使用） |
0x05 | LC_UNIXTHREAD | load_unixthread | 开启一个unix线程（初始化栈布局和寄存器）通常情况下，除了指令指针/程序计数器之外，大部分的寄存器值都为0，指令指针寄存器除外。该命令后续被LC_MAIN替换
0x1D | LC_CODE_SIGNATURE | laod_code_signature | 代码签名 |
0x21 | LC_ENCRYPTION_INFO | set_code_unprotect | 加密的二进制文件 |

- #### LC_SEGMENT/LC_SEGMENT_64参数

参数 | 用途 |
---- | ---- |
segname | load_segment|
vmaddr  | 所描述的段的虚拟物理地址 |
vmsize  | 为该段分配的虚拟内存大小 |
fileoff | 该段在文件中的偏移量 |
filesize | 该段在文件中占用的字节数 |
maxport | 段的页面所需要的最高内存保护，用8进制表示（4 = r， 2 = w，1 = x）
initprot | 段的页面初始内存保护 |
nsects | 段中的section数量 |
flags  | 标志位 |

含义：从偏移量为fileoff处加载filesize字节到虚拟内存地址vmaddr处开始的vmsize字节。每个段的页面根据initprot进行初始化。initprot指定了页面初始化保护的级别（可读/可写/可执行），短的保护设置可以动态改变，但不能超过maxprot指定的值。

段类型 | 含义 |
---- | ---- |
__PAGEZERO | 空指针陷阱 |
__TEXT | 程序代码 |
__DATA | 程序数据 |
__LINKEDIT | 链接器使用的符号和其他表 |

段（segment）可进一步拆解为区（section）

- #### LC_UNIXTHREAD
当所有库完成加载后，dyld的工作也完成了，之后由该命令启动二进制程序的主线程。

- ### LC_THREAD
和LC_UNIXTHREAD类似，用于核心转储文件。

- #### LC_MAIN
设置程序主线程入口点地址和栈大小

- #### LC_CODE_SIGNATURE
OS X中未使用数字签名，iOS中强制使用。如果签名和代码本身不匹配，内核会给进程发送一个SIGKILL信号将进程杀掉。

- #### LC_LOAD_DYLINKER
Mach-o镜像上有很多对外部库和符号的引用，这些要在程序启动时进行绑定，这项工作由动态链接器完成，过程称之为“符号绑定(binding)”

这个加载命令可以启动任何程序作为参数，链接器接管刚创建的进程的控制权，因为内核将进程的入口点设置为链接器的入口。因为库之间存在依赖，因此binding操作需递归完成。
dyld是一个用户态进程，不属于内核的一部分。

dyld处理的加载命令：

\# | 加载命令 | 用途 |
---- | ---- | ---- |
0x02 | LC_SYMTAB | 符号表。符号表和字符串表是由这些命令中指定的偏移量分别提供的
0x0B | LC_DSYMTAB | 同上
0x0C | LC_LOAD_DYLIB | 加载额外的动态库
0x20 | LC_LAZY_LOAD_DULIB | 和LC_LOAD_DYLIB功能一样，将实际的加载工作延迟到第一次使用这个库中的符号时
0x0D | LC_ID_DYLIB | 只在dylib中寻找，指定了dylib的ID、时间戳、版本和兼容版本
0x1F | LC_REEXPORT_DYLIB | 只在动态库中寻找。允许一个库将其他库的符号作为自己的符号重新导出。
0x24 | LC_VERSION_MIN_IPHONEOS | 当前Mach-o要求的最低iOS系统版本
0x25 | LC_VERSION_MIN_MACOSX | 当前Mach-o要求的最低Mac OSX系统版本
0x26 | LC_FUNCTION_STARTS | 压缩的函数起始地址表
0x2A | LC_SOURCE_VERSION | 构建这个二进制文件使用的源代码版本。非正式场合使用，不会对实际的链接产生影响。

如果二进制文件中使用了外部定义的函数和符号，则在\__TEXT段会有一个\__stubs(桩)的区，在这个区中存放的是这些本地未定义符号的占位符。编译器生成代码时会创建对符号桩区的调用，链接器在运行时会解决对桩的调用。

链接器的解决办法是在调用的地址处放置一条`jmp`指令，`jmp`指令指向__stub_helper，进而由`dyld_stub_binder`方法进行符号绑定，然后完成方法调用。`fishhook`即是利用这个机制完成系统C方法hook的。

LC_LOAD_DYLIB命令告诉链接器在哪里可以找到这些符号，链接器要加载每一个指定的库，并在库的符号表（符号名称和地址对应）中被搜寻匹配的符号。


## 地址空间分段
- #### __PAGEZERO
32位下是4KB，64位下是4GB。用于捕捉空指针引用，或捕捉将证书当做指针引用。因这个空间的所有读写权限都被撤销，所以这个范围内的任何引用操作都会引发MMU的硬件页错误，进而产生一个内核可以捕捉的陷阱，内核将这个陷阱转换为C++异常或表示总线错误的POSIX信号（SIGBUS）

- #### __TEXT
存放程序代码，可读可执行权限，不可修改。

- #### __LINKEDIT
由dyld使用，这个区包含字符串表、符号表以及其他数据

- #### __DATA
可读可写的数据

## Data区
```C
#define    SEG_PAGEZERO    "__PAGEZERO"     // 当是 MH_EXECUTE 文件时，捕获到空指针
#define    SEG_TEXT        "__TEXT"         // 代码/只读数据段
#define    SEG_DATA        "__DATA"         // 数据段
#define    SEG_OBJC        "__OBJC"         // Objective-C runtime用到的一些数据
#define    SEG_LINKEDIT    "__LINKEDIT"     // 包含需要被动态链接器使用的符号表、字符串表、重定位表等

// 加载命令结构体，所有内容通过加载命令加载
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
`__TEXT.__text` | 程序代码，**只读**
`__TEXT.__cstring` | 去重后的c字符串
`__DATA.__cfstring` | 程序中使用的 Core Foundation 字符串（CFStringRefs）
`__TEXT.__const` | const关键字修饰的常量
`__TEXT.__unwind_info` | 存储异常处理信息
`__TEXT.__eh_frame` | 调试辅助信息
`__TEXT.__objc_methname` | 方法名称
`__TEXT.__objc_methtype` | OC方法类型
`__TEXT.__objc_classname` | 类名称
`__TEXT.__stubs` | **用于 stub 的占位代码，很多地方称之为桩代码。本质上是一小段会直接跳入lazy binding的表（`__DATA.__la_symbol_ptr`）对应项指针指向的代码。**
`__TEXT.__stubs_helper` | **当 stub 无法找到真正的符号地址后的最终指向**
-- |
`__DATA.__data` | 初始化过的数据，**可读可写**
`__DATA.__la_symbol_ptr` | **lazy binding的指针表，表中的指针一开始都指向__stub_helper，函数名保存在`Dynamic Symbol table`表的对应项里**
`__DATA.__nl_symbol_ptr` | **非lazy binding的指针表，每个表项中的指针都指向一个在装载过程中被动态链机器搜索完成的符号**
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


在mach-o中，`__text`段的内容是只读的，`__data`段的内容是可读可写的，因此`__data`段中的内容是可以修改的，`fishhook`即是通过修改`__data`段达到hook系统c函数的目的。

编译mach-o文件时会预留一些空间作为符号表，存放在`__data`段

工程中所有引用了系统共享缓存中的系统库方法，其指向都设置成符号地址

当dyld将应用加载到内存，或在调用相关方法时，会根据符号表做方法绑定
