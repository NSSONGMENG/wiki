依据objc4-779.1

## map_images_nolock粗解

```C
void map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    mutex_locker_t lock(runtimeLock);
    return map_images_nolock(count, paths, mhdrs);
}
```

`map_images`函数中有两个操作，加锁，调用`map_images_nolock()`函数。
`map_images_nolock()`函数中经过一些列操作后，调用了`read_images()`函数。

> 注：为方便查看加载过程，以下代码对源码各主要抽象方法进行了合并，并忽略了源码的诸多细节

```C
/// @param mhCount     Mach-o Header的数量
/// @param mhPaths[]   Mach-o Header的路径数组，元素数与mhCount一致
/// @param mhdrs[]     Mach-o Header数组，元素数与mhCount一致，元素为mach_header结构体
map_images_nolock(unsigned mhCount, const char * const mhPaths[],
                  const struct mach_header * const mhdrs[]) {
    header_info * hList[mhCount];

    // 1. 初始化dyld预优化共享缓存
    preopt_init();

    // 2. 逆序遍历，处理mach_header
    uint32_t i = mhCount;
    while (i--) {
        // 2.1. 追加至mach_hader单向链表（由底层到表层，从libdispatch.dylib、CoreFoundation到应用程序）
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

主要步骤如上，下面从第一步开始分析主要步骤：
### 步骤1： `preopt_init()`
