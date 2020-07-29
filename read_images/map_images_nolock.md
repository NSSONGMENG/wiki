依据objc4-779.1

## map_images_nolock粗解

`map_images`顾名思义是处理镜像映射的操作

```C
void map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    mutex_locker_t lock(runtimeLock);
    return map_images_nolock(count, paths, mhdrs);
}
```

`map_images`函数中有两个操作，1. 加锁  2.调用`map_images_nolock()`函数

`map_images_nolock()`函数中经过一些列操作后，调用了`read_images()`函数处理镜像。

> 注：为方便查看加载过程，以下代码忽略了源码的诸多细节

```C
///
///
/// @param mhCount     Mach-o Header的数量
/// @param mhPaths[]   Mach-o Header的路径数组，元素数与mhCount一致
/// @param mhdrs[]     Mach-o Header数组，元素数与mhCount一致，元素为mach_header结构体
map_images_nolock(unsigned mhCount, const char * const mhPaths[],
                  const struct mach_header * const mhdrs[]) {
    header_info * hList[mhCount];

    // 1. 初始化dyld预优化共享缓存
    preopt_init();

    // 2. 逆序遍历，处理mach_header，形成一个mach_header链表（由底层到表层，从libdispatch.dylib、CoreFoundation到应用程序）
    // 同时，将生成可读写的hader_info，保存在hList数组中，为read_images操作做准备
    uint32_t i = mhCount;
    while (i--) {
        // 2.1. 追加至mach_hader单向链表
        auto hi = addHeader(mhdr, mhPaths[i], totalClasses, unoptimizedTotalClasses);

        // 2.2 要求分页可执行文件
        if (mhdr->filetype == MH_EXECUTE) {
            // 计算选择器引用的数量
            size_t count;
            _getObjc2SelectorRefs(hi, &count);
            selrefCount += count;
            _getObjc2MessageRefs(hi, &count);
            selrefCount += count;
        }

        hList[hCount++] = hi;
    }

    // 3. 初始化内部使用的选择器表，并注册构造和析构选择器
    sel_init(selrefCount);

    // 4. 初始化自动释放池page，SideTablesMap，关联表（runtime保存关联对象的表？）
    arr_init();

    // 5. 加载镜像
    _read_images(hList, hCount, totalClasses, unoptimizedTotalClasses);

    // 6. 所有镜像数据准备好之后调用加载器加载镜像
    // loadImageFuncs为空，暂不清楚咋回事
    for (auto func : loadImageFuncs) {
        for (uint32_t i = 0; i < mhCount; i++) {
            func(mhdrs[i]);
        }
    }
}
```


### 存疑：
- dyld共享缓存的作用
- 生成的mach_header链表用处，以及为何从底层到表层开始遍历
- 选择器表的作用
- SideTablesMap和关联表的作用

[read_images](https://github.com/NSSONGMENG/wiki/blob/master/read_images/_read_images.md)文件已经准备好了，最近加班开撸~

迷雾重重，几乎对每一步都有存疑😹
