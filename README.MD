
# iOS源码分析

## 可执行文件装载3步曲
1. 创建一个独立的虚拟地址空间（虚拟空间到物理内存的映射）
> **虚拟空间是由一组页映射函数将虚拟空间的各个页映射至物理空间**，创建虚拟空间实际上是创建映射函数所需要的数据结构（页目录的分配），甚至不设置页映射关系，这些映射关系等到后面程序发生页错误的时候进行设置。

2. 读取可执行文件头，并建立虚拟地址空间与可执行文件的映射关系（虚拟空间到可执行文件的映射）
> 映射关系只是保存在操作系统内部的一个数据结构。建立虚拟空间到可执行文件的映射关系后，当程序发生页错误时，操作系统将从物理内存中分配一个物理页，然后通过该映射关系（对应的数据结构）计算出该”缺页”在可执行文件中的相对位置，并将该页从磁盘中读进内存，再设置该虚拟页和物理页的映射关系。至此完成缺页的加载过程。
>
> 根据可执行文件内不同内容的读写执行等属性的不同，将可执行文件分为不同的段，Linux中将虚拟空间中的一个段叫做 **虚拟内存区域（VMA virtual memory area）**，windows中叫做 **虚拟段（virtual section）**。
>
> 可执行文件被映射是以系统的页长度为单位进行，每个段在映射时都应是页长度的整数倍。即便一个段达不到一个页长度，也会占用一个页的空间，即内存对齐。
>
> 在加载可执行文件时，主要关心段的读写执行权限，因此对于权限相同的段可以合并到一起当做一个段(`section`)，有助于加载时进行内存对齐，减少页面碎片，达到节省内存的目的。
>
> 具有相同属性的几个段组成`segment`，因此可以说可执行文件中其实只有为数不多的几个`segment`，各`segment`之间的权限互不相同：
> 1. 代码段为代表的可读可执行的segment
> 2. 数据段和BSS段为代表的可读可写的segment
> 3. 只读数据段为代表的只读的segment
>
> 每个`segment`映射进虚拟内存后对应一个`VMA`，编译是以`section`划分的，加载是以`section`划分的


3. 将CPU的指令寄存器设置为可执行文件的入口地址，启动加载过程。

##### [objc4-779.1代码](https://github.com/NSSONGMENG/objc4-779.1.git)

- #### [mach-o结构](https://github.com/NSSONGMENG/wiki/tree/master/mach-o/mach-o.md)
- #### [_objc_init粗解](https://github.com/NSSONGMENG/wiki/tree/master/pre_main/_objc_init.md)
- #### [_read_images粗解](https://github.com/NSSONGMENG/wiki/tree/master/pre_main/read_images/_read_images.md)

![mach-o.jpg](https://github.com/NSSONGMENG/wiki/blob/master/images/_objc_init.jpg)
