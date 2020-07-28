## 依据objc4-779.1

![mach-o.jpg](https://github.com/NSSONGMENG/wiki/blob/master/images/_objc_init.png)
[思维导图文件](https://github.com/NSSONGMENG/wiki/blob/master/mindmaster/_objc_init.emmx)

`main`函数执行之前，系统会加载mach-o文件，这个过程以`_objc_init()`函数开始，载入类、分类、协议等，为程序的运行做准备。

下面简要分析`_objc_init()`都干了哪些事

```Objective-C
_objc_init() {
    // 注册处理函数
    _dyld_objc_notify_register() {
        // 1. 处理镜像映射
        map_images(){
            map_images_nolock(){
                // 1.1. 找出所有OC元数据镜像
                // 1.2. 读取镜像，下方给出过程
                _read_images() {}
                // 1.3. 调用加载器加载镜像 ？
           }

        },
        // 2. 加载镜像
        load_images(){}
        // 卸载镜像
        unload_images(){}
    }
}
```
## read_images操作
```C
_read_images() {
  // 修复selector引用
  for (EACH_HEADER) {
      SEL * sels = _getObjc2SelectorRefs(hi, &count);
  }
  // 读取类
  for (EACH_HEADER) {
      // 类列表
      classref_t const * classlist = _getObjc2ClassList(hi, &count);
      // 类
      Class newCls = readClass(cls, headerIsBundle, headerIsPreoptimized);
  }
  // class重映射
  for (EACH_HEADER) {
      Class * classrefs = _getObjc2ClassRefs(hi, &count);
      remapClassRef(Class)
  }
  // 读取协议，修复协议引用
  for (EACH_HEADER) {
      protocol_t * const * protolist = _getObjc2ProtocolList(hi, &count);
  }
  // 修复协议引用
  for (EACH_HEADER) {
      protocol_t ** protolist = _getObjc2ProtocolRefs(hi, &count);
  }
  // 加载分类
  for (EACH_HEADER) {
      category_t * const * catlist = _getObjc2CategoryList(hi, $count)
      category_t * cat = catlist[i];
  }
  // 实现非懒加载类，为+load调用和静态变量初始计划做准备
  for (EACH_HEADER) {
    classref_t const * classlist = _getObjc2NonlazyClassList(hi, &count);
    addClassTableEntry(cls);
  }
}
```

## load_images操作
这一步操作的目的是调用类和分类的+load方法

- 准备+load方法列表
    - 处理类的+load，递归处理从父类开始添加，放入loadable_classes数组
    - 处理分类的+load，放入loadable_categories数组
- 调用+load方法
    - 调用类的+load
    - 调用分类的+load

```C

// 处理+load方法使用，保存类和其+load
struct loadable_class {
    Class cls;  // may be nil
    IMP method;
};

// 处理+load方法使用，保存分类和其+load
struct loadable_category {
    Category cat;  // may be nil
    IMP method;
};

// 为方便查看处理过程，以下代码对源码各抽象方法进行了合并，并忽略了源码的诸多细节
load_images(){
    // 1. 准备+load方法，将+load方法放入loadable_classes数组
    prepare_load_methods(){
        // 1.1. 处理类的+load
        classref_t const * classlist = _getObjc2NonlazyClassList(mhdr, &count);
        for (i = 0; i < count; i++) {
            // 处理类的+load方法
            schedule_class_load(remapClass(classlist[i])){
                // 若类已加处理，则退出
                if (cls->data()->flags & RW_LOADED) return;

                // 【重点细节】
                // 递归处理继承链的+load，从根类的+load开始添加
                schedule_class_load(cls->superclass);

                add_class_to_loadable_list(cls);
                // 写入已加载标记
                cls->setInfo(RW_LOADED);
            }
        }

        // 1.2. 处理分类的+load，分类的+load方法放入loadable_categories数组
        category_t * const * categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
        for (i = 0; i < count; i++) {
            category_t * cat = categorylist[i];
            Class cls = remapClass(cat->cls);
            // 添加category里的+load
            add_category_to_loadable_list(cat);
        }
    }

    // 2. 调用+load方法
    call_load_methods(){
        // 在自动释放池中调用+load
        void * pool = objc_autoreleasePoolPush();

        //【细节】之所以使用do while循环，应该是考虑到在处理+load方法时可能存在多线程处理的情况，
        // 即在执行do-while循环时可能存在新的+load方法被发现和添加的情况
        do {
            // 2.1. 重复调用类的+load，知道所有的类+load都被调用
            while (loadable_classes_used > 0) {
                call_class_loads();
            }

            // 2.2. 调用分类的+load
            more_categories = call_category_loads();

            // 2.3. 如果还有其他的类和分类，则继续循环
        } while (loadable_classes_used > 0  ||  more_categories);

        objc_autoreleasePoolPop(pool);
    }
}
```
