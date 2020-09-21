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

* 有关代码中的`fastpath` 和 `slowpath`

```  
//x很可能为真， fastpath 可以简称为 真值判断  
#define fastpath(x) (__builtin_expect(bool(x), 1))   
//x很可能为假，slowpath 可以简称为 假值判断  
#define slowpath(x) (__builtin_expect(bool(x), 0)) 
```  

其中的__builtin_expect指令是由gcc引入的  

1. 目的：编译器可以对代码进行优化，以减少指令跳转带来的性能下降。即性能优化  
2. 作用：允许程序员将最有可能执行的分支告诉编译器。  
3. 指令的写法为：__builtin_expect(EXP, N)。表示 EXP==N的概率很大。  
4. fastpath定义中__builtin_expect((x),1)表示 x 的值为真的可能性更大；即 执行if 里面语句的机会更大  
5. slowpath定义中的__builtin_expect((x),0)表示 x 的值为假的可能性更大。即执行 else 里面语句的机会更大  
6. 在日常的开发中，也可以通过设置来优化编译器，达到性能优化的目的，设置的路径为：Build Setting --> Optimization Level --> Debug --> 将None 改为 fastest 或者 smallest  


### objc源码  

访问参考  
[苹果开源源码汇总](https://opensource.apple.com)  
[更直接的地址](https://opensource.apple.com/tarballs/)  

### 底层调试 (持续更新)  

1. 符号断点 (断点中增加 Symbolic Breakpoint, 然后增加要跟踪的函数名, 比如alloc, objc_alloc等内部函数名)
2.  按住`control` + `step into`
3.  汇编跟流程

![全局汇编断点](https://upload-images.jianshu.io/upload_images/1367029-75bb8ca960109fc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)