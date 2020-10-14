在上篇文章[iOS 程序加载流程分析](https://www.jianshu.com/p/a6d46a55aa89)中分析dyld的过程中, 其中有一幅图来分析_objc_init符号断点图, 如下:  

![1419656-0cb5ee83eefa4ac6.png](https://upload-images.jianshu.io/upload_images/1367029-4069fb7f8e27e57f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

结合这张图我们得知`_objc_init`调用的流程大致为:  
`dyld`的`doModInitFunctions`方法调用`libSystem.B.dylib的libSystem_initializer`方法；接着初始化了`libdispatch; libdispatch`又调用了`_os_object_int`,最终来到了`_objc_init`.

下面我们来看看`_objc_init`的官方源码(objc4-781)  

```
void _objc_init(void)  
{  
    static bool initialized = false;  
    if (initialized) return;  
    initialized = true;  
    
    // fixme defer initialization until an objc-using image is found?  
    environ_init();  		//读取影响运行时的环境变量。如果需要，还可以打印环境变量帮助.  
    tls_init();  				//关于线程key的绑定一比如每线程数据的析构函数.  
    static_init();  			//运行C ++静态构造函数。在dyld调用我们的静态构造函数之前, 'libc'会调用_objc_inint(), 因此我们必须自己做. 
    runtime_init();  		//runtime运行时环境初始化，里面主要是: unattachedCategories ， allocatedClasses后面会分析
    exception_init();  		//异常信息的初始化
    cache_init();  			//缓存条件初始化
    _imp_implementationWithBlock_init();  		//启动回调机制。通常这不会做什么，因为所有的初始化都是情性的，但是对于某些进程，我们会迫不及待地加载trampolines dylib。


    // 什么时候调用? images 镜像文件  
    // map_images()  
    // load_images()  
    
    _dyld_objc_notify_register(&map_images, load_images, unmap_image);  

#if __OBJC2__  
    didCallDyldNotifyRegister = true;  
#endif  
}  
```

#### 1. 环境变量初始化(environ_init)
在不设置环境变量 `OBJC_DISABLE_NONPOINTER_ISA` 的时候，打印 person 的 isa 信息, 如下:  

```
lldb) x/4gx person
0x1010b5680: 0x001d800100008265 0x0000000000000000
0x1010b5690: 0x0000000000000000 0x0000000000000000
(lldb) p/t 0x001d800100008265
(long) $1 = 0b0000000000011101100000000000000100000000000000001000001001100101
```

然后设置环境变量`OBJC_DISABLE_NONPOINTER_ISA` 为YES之后, 再次打印结果如下:  

![设置环境变量OBJC_DISABLE_NONPOINTER_ISA 为 YES](https://upload-images.jianshu.io/upload_images/1367029-c192cd20769aa5d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
(lldb) x/4gx person
0x100a09f20: 0x0000000100008260 0x0000000000000000
0x100a09f30: 0x0000000000000000 0x0000000000000000
(lldb) p/t 0x0000000100008260
(long) $1 = 0b0000000000000000000000000000000100000000000000001000001001100000
```

我们可以看到最后一位发生了变化, 而在[isa结构分析](https://www.jianshu.com/p/b2053bdae5a2)这篇文章中, 我们可以得知,最后一位就是 `nonpointer` 位，表示是否对 isa 指针开启指针优化。 `0`：纯 isa 指针；`1`：不止是类对象地址。isa 中包含了类信息、对象的引用计数等。

#### 2.  _dyld_objc_notify_register
其实我们是要重点分析的这里, 这里是跨库调用的, 源码在`dyld`的源码里.  如下是`dyld`的源码部分:  

```
void registerObjCNotifiers(_dyld_objc_notify_mapped mapped, _dyld_objc_notify_init init, _dyld_objc_notify_unmapped unmapped)
{  
     // record functions to call  
     sNotifyObjCMapped    = mapped;  			//map_images  
     sNotifyObjCInit        = init;  				//load_images  
     sNotifyObjCUnmapped = unmapped;  		//unmap_image  


     // call 'mapped' function with all images mapped so far  
     try {  
         notifyBatchPartial(dyld_image_state_bound, true, NULL, false, true);  
     }  
     catch (const char* msg) {  
         // ignore request to abort during registration  
     }  


     // <rdar://problem/32209809> call 'init' function on all images already init'ed (below libSystem)  
     for (std::vector<ImageLoader*>::iterator it=sAllImages.begin(); it != sAllImages.end(); it++) {  
         ImageLoader* image = *it;  
         if ( (image->getState() == dyld_image_state_initialized) && image->notifyObjC() ) {  
             dyld3::ScopedTimer timer(DBG_DYLD_TIMING_OBJC_INIT, (uint64_t)image->machHeader(), 0, 0);  
             (*sNotifyObjCInit)(image->getRealPath(), image->machHeader());  // 调用了load_image
         }  
     }  
}  

```

1. 这就是`_objc_init`方法里调用了`_dyld_objc_notify_register `方法在`dyld`源码中找到了真正调用的地方.  
2. 同样我们在`dyld`中也能找到`sNotifyObjCMapped `的调用.  

#### 3. 有关map_images解析

```
void  
map_images(unsigned count, const char * const paths[],  
           const struct mach_header * const mhdrs[])  
{  
    mutex_locker_t lock(runtimeLock);  
    return map_images_nolock(count, paths, mhdrs);  
}  
```
接着进入`map_images_nolock`函数看看，这里的核心代码是`_read_images`方法  

```
    if (hCount > 0) {  
        _read_images(hList, hCount, totalClasses, unoptimizedTotalClasses);  
    }  
```

1. 条件控制进⾏⼀次的加载
2. 修复预编译阶段的 @selector 的混乱问题
3. 错误混乱的类处理
4. 修复重映射⼀些没有被镜像⽂件加载进来的 类
5. 修复⼀些消息!
6. 当我们类⾥⾯有协议的时候 : readProtocol
7. 修复没有被加载的协议
8. 分类处理
9. 类的加载处理
10. 没有被处理的类 优化那些被侵犯的类

#### 4. 有关readClass解析

```
Class readClass(Class cls, bool headerIsBundle, bool headerIsPreoptimized)  
{  
    const char *mangledName = cls->mangledName();  
    
    if (missingWeakSuperclass(cls)) {  
        // No superclass (probably weak-linked).   
        // Disavow any knowledge of this subclass.  
        if (PrintConnecting) {  
            _objc_inform("CLASS: IGNORING class '%s' with "  
                         "missing weak-linked superclass",   
                         cls->nameForLogging());  
        }  
        addRemappedClass(cls, nil);  
        cls->superclass = nil;  
        return nil;  
    }  
    
    cls->fixupBackwardDeployingStableSwift();  

    Class replacing = nil;  
    if (Class newCls = popFutureNamedClass(mangledName)) {  
        // This name was previously allocated as a future class.  
        // Copy objc_class to future class's struct.  
        // Preserve future's rw data block.  
        
        if (newCls->isAnySwift()) {  
            _objc_fatal("Can't complete future class request for '%s' "  
                        "because the real class is too big.",   
                        cls->nameForLogging());  
        }  
        
        class_rw_t *rw = newCls->data();  
        const class_ro_t *old_ro = rw->ro();  
        memcpy(newCls, cls, sizeof(objc_class));  
        rw->set_ro((class_ro_t *)newCls->data());  
        newCls->setData(rw);  
        freeIfMutable((char *)old_ro->name);  
        free((void *)old_ro);  
        
        addRemappedClass(cls, newCls);  
        
        replacing = cls;  
        cls = newCls;  
    }  
    
    if (headerIsPreoptimized  &&  !replacing) {  
        // class list built in shared cache  
        // fixme strict assert doesn't work because of duplicates  
        // ASSERT(cls == getClass(name));  
        ASSERT(getClassExceptSomeSwift(mangledName));  
    } else {  
        addNamedClass(cls, mangledName, replacing);  
        addClassTableEntry(cls);  
    }  

    // for future reference: shared cache never contains MH_BUNDLEs  
    if (headerIsBundle) {  
        cls->data()->flags |= RO_FROM_BUNDLE; 
        cls->ISA()->data()->flags |= RO_FROM_BUNDLE;  
    }  
    
    return cls;  
}  
```

这一步会把class信息从二进制里面读出来, 然后:  

1. 将`newCls->data()`取出来作`rw`.
2. 将`newCls->data(`)再取出来强转为`class_ro_t *`放到到`rw`的`ro`部分.
3. `addClassTableEntry`这是将类插入到类的集合表中，为了后面调用的快速查找.  

文章参考:  
[dyld和ObjC的关联](https://www.jianshu.com/p/3cad4212892a)
[dyld和ObjC的关联](https://www.jianshu.com/p/f928adc138b4)