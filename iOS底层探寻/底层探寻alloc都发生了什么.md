###有关alloc之后, 底层代码的执行顺序以及解析  
![alloc流程图](https://upload-images.jianshu.io/upload_images/1367029-d9fa77e3da21c476.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 从上面的流程图可以很清晰的看到alloc之后的执行方法顺序  
2. 最重要的就是最后的三个方法 `instanceSize` `calloc` 以及`obj->initInstanceIsa`

* instanceSize: 先计算出要开辟多大内存空间  
* calloc: 开辟内存空间, 申请内存, 返回地址指针
* obj->initInstanceIsa: 关联到相应的类  

以下放上源码参考:   

```  
static ALWAYS_INLINE id  
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)  
{  
    ASSERT(cls->isRealized());  

    // Read class's info bits all at once for performance  
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();  
    bool hasCxxDtor = cls->hasCxxDtor();  
    bool fast = cls->canAllocNonpointer();  
    size_t size;  
    // 1:要开辟多少内存  
    size = cls->instanceSize(extraBytes);  
    if (outAllocatedSize) *outAllocatedSize = size;  

    id obj;  
    if (zone) {  
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);  
    } else {  
        // 2;怎么去申请内存  
        obj = (id)calloc(1, size);  
    }  
    if (slowpath(!obj)) {  
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {  
            return _objc_callBadAllocHandler(cls);  
        }  
        return nil;  
    }  

    // 3: ?
    if (!zone && fast) {  
        obj->initInstanceIsa(cls, hasCxxDtor);  
    } else {  
        // Use raw pointer isa on the assumption that they might be  
        // doing something weird with the zone or RR.  
        obj->initIsa(cls);  
    }  

    if (fastpath(!hasCxxCtor)) {  
        return obj;  
    }  

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;  
    return object_cxxConstructFromClass(obj, cls, construct_flags);  
}  
```  

* 有关代码中的`fastpath` 和 `slowpath`, 我的理解是DeBug编译模式和Release编译模式, Release模式走的就是fastpath来优化编辑的


### objc源码  

访问参考  
[苹果开源源码汇总](https://opensource.apple.com)  
[更直接的地址](https://opensource.apple.com/tarballs/)  

### 底层调试 (持续更新)  

![全局汇编断点](https://upload-images.jianshu.io/upload_images/1367029-75bb8ca960109fc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)