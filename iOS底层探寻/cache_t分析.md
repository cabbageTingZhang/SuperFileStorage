紧接着我们来分析类结构体中`cache_t`, 只从单词来看就能猜出来是与缓存有关. 下面我们先看cache_t的源码:   

```  
struct cache_t {  
#if 1 // Mac   
    struct bucket_t * _buckets;  
    mask_t _mask;  
#elif 1 // 真机  (尽量用真机调试, 因为真机更贴近日常使用)
    uintptr_t _maskAndBuckets;  
    mask_t _mask_unused;  
    
    // How much the mask is shifted by.  
    static constexpr uintptr_t maskShift = 48;  
    
    // Additional bits after the mask which must be zero. msgSend  
    // takes advantage of these additional bits to construct the value  
    // `mask << 4` from `_maskAndBuckets` in a single instruction.  
    static constexpr uintptr_t maskZeroBits = 4;  
    
    // The largest mask value we can store.  
    static constexpr uintptr_t maxMask = ((uintptr_t)1 << (64 - maskShift)) - 1;  
    
    // The mask applied to `_maskAndBuckets` to retrieve the buckets pointer.  
    static constexpr uintptr_t bucketsMask = ((uintptr_t)1 << (maskShift - maskZeroBits)) - 1;  
#endif  
    uint16_t _flags; // 标志位  
    uint16_t _occupied; // 被占用的  

public:
    static bucket_t *emptyBuckets();  
    struct bucket_t *buckets();  
    mask_t mask();  
    mask_t occupied();  
    void incrementOccupied();  
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);  
    void initializeToEmpty();  
    unsigned capacity();  
    bool isConstantEmptyCache();  
    bool canBeFreed();  
    void reallocate(mask_t oldCapacity, mask_t newCapacity, bool freeOld);  
    void insert(Class cls, SEL sel, IMP imp, id receiver);  
};  

```

在进行结构分析时, 我们先写了一个例子来分析`cache_t`的结构. 具体例子如下: 

```
//LGPerson.h文件  
@interface LGPerson : NSObject  

@property (nonatomic, copy) NSString *lgName;  
@property (nonatomic, strong) NSString *nickName;  

- (void)say1;  
- (void)say2;  
- (void)say3;  
- (void)say4;   

//LGPerson.m文件  
#import "LGPerson.h"  

@implementation LGPerson  

- (void)say1{  
    NSLog(@"LGPerson say : %s",__func__);  
}  
- (void)say2{  
    NSLog(@"LGPerson say : %s",__func__);  
}  
- (void)say3{  
    NSLog(@"LGPerson say : %s",__func__);  
}  
- (void)say4{  
    NSLog(@"LGPerson say : %s",__func__);  
}  

@end  

``` 

具体调用以及打印结果, 见下方代码:  

```  
#import <Foundation/Foundation.h>  
#import "LGPerson.h"  
#import <objc/runtime.h>  

typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits  

struct lg_bucket_t {  
    SEL _sel;  
    IMP _imp;  
};  
 
struct lg_cache_t {  
    struct lg_bucket_t * _buckets;  
    mask_t _mask;  
    uint16_t _flags;  
    uint16_t _occupied;  
};  

struct lg_class_data_bits_t {  
    uintptr_t bits;  
};  

struct lg_objc_class {  
    Class ISA;  
    Class superclass;  
    struct lg_cache_t cache;             // formerly cache pointer and vtable  
    struct lg_class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags  
};  


int main(int argc, const char * argv[]) {  
    @autoreleasepool {  
        LGPerson *p  = [LGPerson alloc];  
        Class pClass = [LGPerson class];  // objc_clas  
        [p say1];  
        [p say2];  
//        [p say3];  
//        [p say4];  
        
        struct lg_objc_class *lg_pClass = (__bridge struct lg_objc_class *)(pClass);  
        NSLog(@"%hu - %u",lg_pClass->cache._occupied,lg_pClass->cache._mask);  
        for (mask_t i = 0; i<lg_pClass->cache._mask; i++) {  
            // 打印获取的 bucket  
            struct lg_bucket_t bucket = lg_pClass->cache._buckets[i];  
            NSLog(@"%@ - %p",NSStringFromSelector(bucket._sel),bucket._imp);  
        }  
        NSLog(@"Hello, World!");  
    }  
    return 0;  
}  

//当屏蔽 [p say3];   [p say4];  方法时的打印结果:   
2020-09-17 21:23:31.259617+0800 003-cache_t脱离源码环境分析[3956:50598] LGPerson say : -[LGPerson say1]  
2020-09-17 21:23:31.259990+0800 003-cache_t脱离源码环境分析[3956:50598] LGPerson say : -[LGPerson say2]  
2020-09-17 21:23:31.260029+0800 003-cache_t脱离源码环境分析[3956:50598] 2 - 3  
2020-09-17 21:23:35.029171+0800 003-cache_t脱离源码环境分析[3956:50598] say1 - 0x2828  
2020-09-17 21:23:35.029352+0800 003-cache_t脱离源码环境分析[3956:50598] say2 - 0x2818  
2020-09-17 21:23:35.029409+0800 003-cache_t脱离源码环境分析[3956:50598] (null) - 0x0  

//当打开屏幕 [p say3];   [p say4]; 的打印结果为:  
2020-09-17 21:37:09.315693+0800 003-cache_t脱离源码环境分析[4296:55622] LGPerson say : -[LGPerson say1]  
2020-09-17 21:37:09.315964+0800 003-cache_t脱离源码环境分析[4296:55622] LGPerson say : -[LGPerson say2]  
2020-09-17 21:37:09.316001+0800 003-cache_t脱离源码环境分析[4296:55622] LGPerson say : -[LGPerson say3]  
2020-09-17 21:37:09.316076+0800 003-cache_t脱离源码环境分析[4296:55622] LGPerson say : -[LGPerson say4]  
2020-09-17 21:37:09.316099+0800 003-cache_t脱离源码环境分析[4296:55622] 2 - 7  
2020-09-17 21:37:09.316196+0800 003-cache_t脱离源码环境分析[4296:55622] say4 - 0x29a8  
2020-09-17 21:37:09.316217+0800 003-cache_t脱离源码环境分析[4296:55622] (null) - 0x0  
2020-09-17 21:37:09.316283+0800 003-cache_t脱离源码环境分析[4296:55622] say3 - 0x29d8  
2020-09-17 21:37:09.316315+0800 003-cache_t脱离源码环境分析[4296:55622] (null) - 0x0   
2020-09-17 21:37:09.316330+0800 003-cache_t脱离源码环境分析[4296:55622] (null) - 0x0 
2020-09-17 21:37:09.316366+0800 003-cache_t脱离源码环境分析[4296:55622] (null) - 0x0  
2020-09-17 21:37:09.316382+0800 003-cache_t脱离源码环境分析[4296:55622] (null) - 0x0  

我们先抛出问题:以及最终的得出的结论        
	  // _occupied  _mask 是什么  cup - 1  
        // 会变化 2-3 -> 2-7   (说明有做扩容操作)
        // bucket 会有丢失  重新申请  
        // 顺序有点问题  哈希  
        // 当打开屏蔽方法后, 没有打印出 say1 say2, 证明有做刷新释放操作
        
```  

然后我们从`cache_t`的源码来分析:  每当对象调用一个方法时, 如果在cache里面没有找到, 就会insert一条缓存:  

```
void cache_t::insert(Class cls, SEL sel, IMP imp, id receiver)  
{  
    // Use the cache as-is if it is less than 3/4 full  
    mask_t newOccupied = occupied() + 1;  
    unsigned oldCapacity = capacity(), capacity = oldCapacity;  
    // 1. 如果Cache 是空的话，会初始化一个 4 个字节的空间  
    if (slowpath(isConstantEmptyCache())) {  
        // Cache is read-only. Replace it.  
        if (!capacity) capacity = INIT_CACHE_SIZE;  
        reallocate(oldCapacity, capacity, /* freeOld */false);  
    }  
    else if (fastpath(newOccupied + CACHE_END_MARKER <= capacity / 4 * 3)) {  
        // Cache is less than 3/4 full. Use it as-is.  
        // 2. newOccupied + CACHE_END_MARKER <= capacity / 4 * 3 ，直接插入  
    }  
    else {  
        // 3. 否则会扩容,  扩容为原空间的 2倍大小
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;  
        if (capacity > MAX_CACHE_SIZE) {  
            capacity = MAX_CACHE_SIZE;  
        }  
        reallocate(oldCapacity, capacity, true);  // // 重新分配空间   存储新的数据，抹除已有缓存
    }  
    // 4. 初始化一个指针数组  
    bucket_t *b = buckets();  
    // 5. 设置掩码为 capacity - 1  
    mask_t m = capacity - 1;  
    // 6. 根据sel 计算 hash 值  
    mask_t begin = cache_hash(sel, m);  
    mask_t i = begin;  
    /*  
     *  扫描第一个未使用的插槽，并且插入   
     */  
    // Scan for the first unused slot and insert there.  
    // There is guaranteed to be an empty slot because the  
    // minimum size is 4 and we resized at 3/4 full.  
    do {  
        // 7. 当前 插槽 取到 的sel 地址为0， 那么插入新的值  
        if (fastpath(b[i].sel() == 0)) {  
            // 8. 增加占用字段并且插入  
            incrementOccupied();  
            b[i].set<Atomic, Encoded>(sel, imp, cls);  
            return;  
        }  
        // 8. 多线程做的判断  
        if (b[i].sel() == sel) {  // 如果找到需要缓存的方法，什么都不做，并退出循环  
            // The entry was added to the cache by some other thread  
            // before we grabbed the cacheUpdateLock.  
            return;  
        }  
        // 9. 如果当前位置已有值，那么就找下一个位置  判断不等于初始下标值 begin 是为了将散列表中的数据全部遍历结束，而cache_next( ) 是为了解决哈希冲突而进行的二次哈希.
    } while (fastpath((i = cache_next(i, m)) != begin));  
}  

// 10. hash 算法, 保证不会越界  
static inline mask_t cache_hash(SEL sel, mask_t mask)   
{  
    return (mask_t)(uintptr_t)sel & mask;  
}  

```

![cache_t缓存流程图](https://upload-images.jianshu.io/upload_images/1367029-ad154e6583db49f9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


总结一下cache_t的结构: 

![cache_t结构](https://upload-images.jianshu.io/upload_images/1367029-5dd1280b8e408336.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* _buckets ：是一个散列表，用来存储 缓存方法的 sel 和 imp.
* _mask ：  有2个作用，1: 作为当前可存储的最大容量；2: 作为掩码，取已缓存方法在 _buckets 中的下标.
* _occupied :  _buckets 中 已缓存的方法数量.
