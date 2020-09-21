本文主要是对objc_msgSend的分析理解, 在分析cache_t的流程时, 我们只分析了写入流程, 其实还有一个cache读取流程, 有`objc_msgSend`和`cache_getImp`.  

#### 先来了解一下runtime  

* runtime : 简称运行时。OC就是运行时机制，也就是在运行时候的一些机制，其中最主要的是消息机制。
* 对于C语言，函数的调用在编译的时候会决定调用哪个函数(在编译阶段，C语言调用未实现的函数就会报错)。
* 对于OC的函数，属于动态调用过程，在编译的时候并不能决定真正调用哪个函数，只有在真正运行的时候才会根据函数的名称找到对应的函数来调用( 在编译阶段，OC可以调用任何函数，即使这个函数并未实现，只要声明过就不会报错)。

runtime的使用有以下三种方式, 其三种实现方法与编译层和底层的关系如下:  

1. 通过OC代码, 例如: `[person sayHello]`.  
2. 通过NSObject方法, 例如: `isKindOfClass`.  
3. 通过runtime API, 例如: `class_getInstanceSize`.  

![runtime三种方式以及底层的关系](https://upload-images.jianshu.io/upload_images/1367029-1e5891ae3448f2c4.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中Compiler就是我们了解的编译器, 即LLVM, 例如alloc对应的底层库objc_alloc, runtime system libarary就是底层库.

#### 探索方法的本质

用clang将代码编译成C++, 我们来查看方法的实现:  

```  
//main.m中方法的调用  
LGPerson *person = [LGPerson alloc];  
[person sayNB];  
[person sayHello];  

//👇clang编译后的底层实现  
LGPerson *person = ((LGPerson *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("LGPerson"),   sel_registerName("alloc"));  
((void (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("sayNB"));  
((void (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("sayHello"));  

//我们查看编译后的代码, 发现方法的本质就是消息发送objc_msgSeng, 我们也可以用objc_msgSeng来完成方法的调用, 如下:  
LGPerson *person = [LGPerson alloc];     
objc_msgSend(person,sel_registerName("sayNB"));  
[person sayNB];  

最后发现都可以打印出来LGPerson 的 sayNB方法 输入 666  

```  

以上使用`objc_msgSend `直接来调用方法需要注意:  

1. 直接调用objc_msgSend，需要导入头文件`#import <objc/message.h>`.  
2. 需要将target --> Build Setting -->搜索msg -- 将`enable strict checking of obc_msgSend calls`由YES 改为NO，将严厉的检查机制关掉，否则objc_msgSend的参数会报错.  

#### objc_msgSengSuper探索

我们通过一下方法代码来进行调试:  

```  
@interface LGTeacher : NSObject  
- (void)sayHello;  
@end  

@implementation LGTeacher  
- (void)sayHello{  
    NSLog(@"666");  
}  
@end  

@interface LGPerson : LGTeacher  
- (void)sayHello;  
- (void)sayNB;  
@end  

@implementation LGPerson  
- (void)sayNB{  
    NSLog(@"666");    
}  
@end  

//如上LGPerson类并没有实现sayHello, 而在其父类中却实现了该方法, 我们通过以下代码来测试
LGPerson *person = [LGPerson alloc];  
LGTeacher *teacher = [LGTeacher alloc];  
[person sayHello];  

struct objc_super lgsuper;  
lgsuper.receiver = person; //消息的接收者还是person  
lgsuper.super_class = [LGTeacher class]; //告诉父类是谁  
    
//消息的接受者还是自己 - 父类 - 请你直接找我的父亲  
objc_msgSendSuper(&lgsuper, sel_registerName("sayHello"));  

//打印结果为输出两次: LGTeacher 666

```  

通过以上代码的调试, 我们发现不论是 `[person sayHello]`还是`objc_msgSendSuper`都执行的是父类中`sayHello`的实现，所以这里，我们可以作一个猜测：方法调用，首先是在类中查找，如果类中没有找到，会到类的父类中查找.  

#### objc_msgSend快速查找流程分析

**先来一张总结图片, 图片来源Cooci 你要问我Cooci是谁, 不告诉你**  
![objc_msgSend流程分析.png](https://upload-images.jianshu.io/upload_images/1367029-d7bd04222cef697c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在objc40-781代码中, 全局搜索objc_msgSend, 这里我们只看真机环境arm64, 发现代码是汇编实现, 具体代码如下: 

```  
//---- 消息发送 -- 汇编入口--objc_msgSend主要是拿到接收者的isa信息  
ENTRY _objc_msgSend   
//---- 无窗口  
 UNWIND _objc_msgSend, NoFrame   
    
//---- p0 和空对比，即判断接收者是否存在，其中p0是objc_msgSend的第一个参数-消息接收者receiver  
    cmp p0, #0          // nil check and tagged pointer check   
//---- le小于 --支持taggedpointer（小对象类型）的流程  
#if SUPPORT_TAGGED_POINTERS  
    b.le    LNilOrTagged        //  (MSB tagged pointer looks negative)   
#else  
//---- p0 等于 0 时，直接返回 空  
    b.eq    LReturnZero   
#endif   
//---- p0即receiver 肯定存在的流程  
//---- 根据对象拿出isa ，即从x0寄存器指向的地址 取出 isa，存入 p13寄存器  
    ldr p13, [x0]       // p13 = isa   
//---- 在64位架构下通过 p16 = isa（p13） & ISA_MASK，拿出shiftcls信息，得到class信息  
    GetClassFromIsa_p16 p13     // p16 = class   
LGetIsaDone:  
    // calls imp or objc_msgSend_uncached   
//---- 如果有isa，走到CacheLookup 即缓存查找流程，也就是所谓的sel-imp快速查找流程  
    CacheLookup NORMAL, _objc_msgSend  

#if SUPPORT_TAGGED_POINTERS  
LNilOrTagged:  
//---- 等于空，返回空  
    b.eq    LReturnZero     // nil check   

    // tagged  
    adrp    x10, _objc_debug_taggedpointer_classes@PAGE  
    add x10, x10, _objc_debug_taggedpointer_classes@PAGEOFF  
    ubfx    x11, x0, #60, #4  
    ldr x16, [x10, x11, LSL #3]  
    adrp    x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGE  
    add x10, x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGEOFF  
    cmp x10, x16  
    b.ne    LGetIsaDone  

    // ext tagged  
    adrp    x10, _objc_debug_taggedpointer_ext_classes@PAGE  
    add x10, x10, _objc_debug_taggedpointer_ext_classes@PAGEOFF  
    ubfx    x11, x0, #52, #8  
    ldr x16, [x10, x11, LSL #3]  
    b   LGetIsaDone  
// SUPPORT_TAGGED_POINTERS  
#endif  

LReturnZero:  
    // x0 is already zero  
    mov x1, #0  
    movi    d0, #0  
    movi    d1, #0  
    movi    d2, #0  
    movi    d3, #0  
    ret  

    END_ENTRY _objc_msgSend  
    
    
    
    //通过关键字GetClassFromIsa_p16在arm64中找到实现方法对应如下代码: 
    
.macro GetClassFromIsa_p16 /* src */   
//---- 此处用于watchOS  
#if SUPPORT_INDEXED_ISA   
    // Indexed isa  
//---- 将isa的值存入p16寄存器  
    mov p16, $0         // optimistically set dst = src   
    tbz p16, #ISA_INDEX_IS_NPI_BIT, 1f  // done if not non-pointer isa -- 判断是否是 nonapointer isa  
    // isa in p16 is indexed  
//---- 将_objc_indexed_classes所在的页的基址 读入x10寄存器  
    adrp    x10, _objc_indexed_classes@PAGE   
//---- x10 = x10 + _objc_indexed_classes(page中的偏移量) --x10基址 根据 偏移量 进行 内存偏移  
    add x10, x10, _objc_indexed_classes@PAGEOFF  
//---- 从p16的第ISA_INDEX_SHIFT位开始，提取 ISA_INDEX_BITS 位 到 p16寄存器，剩余的高位用0补充  
    ubfx    p16, p16, #ISA_INDEX_SHIFT, #ISA_INDEX_BITS  // extract index   
    ldr p16, [x10, p16, UXTP #PTRSHIFT] // load class from array  
1:  

//--用于64位系统  
#elif __LP64__   
    // 64-bit packed isa  
//---- p16 = class = isa & ISA_MASK(位运算 & 即获取isa中的shiftcls信息)  
    and p16, $0, #ISA_MASK   

#else  
    // 32-bit raw isa ---- 用于32位系统  
    mov p16, $0  

#endif  

.endmacro  



//通过CacheLookup NORMAL 来找到缓存查找汇编源码:  
//！！！！！！！！！重点！！！！！！！！！！！！  
.macro CacheLookup   
    //  
    // Restart protocol:  
    //  
    //   As soon as we're past the LLookupStart$1 label we may have loaded  
    //   an invalid cache pointer or mask.  
    //  
    //   When task_restartable_ranges_synchronize() is called,  
    //   (or when a signal hits us) before we're past LLookupEnd$1,  
    //   then our PC will be reset to LLookupRecover$1 which forcefully  
    //   jumps to the cache-miss codepath which have the following  
    //   requirements:  
    //  
    //   GETIMP:  
    //     The cache-miss is just returning NULL (setting x0 to 0)  
    //  
    //   NORMAL and LOOKUP:  
    //   - x0 contains the receiver  
    //   - x1 contains the selector  
    //   - x16 contains the isa  
    //   - other registers are set as per calling conventions  
    //  
LLookupStart$1:  

//---- p1 = SEL, p16 = isa --- #define CACHE (2 * __SIZEOF_POINTER__)，其中 __SIZEOF_POINTER__表示pointer的大小 ，即 2*8 = 16  
//---- p11 = mask|buckets -- 从x16（即isa）中平移16字节，取出cache 存入p11寄存器 -- isa距离cache 正好16字节：isa（8字节）-superClass（8字节）-cache（mask高16位 + buckets低48位）  
    ldr p11, [x16, #CACHE]              
//---- 64位真机  
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16   
//--- p11(cache) & 0x0000ffffffffffff ，mask高16位抹零，得到buckets 存入p10寄存器-- 即去掉mask，留下buckets  
    and p10, p11, #0x0000ffffffffffff   // p10 = buckets   
    
//--- p11(cache)右移48位，得到mask（即p11 存储mask），mask & p1(msgSend的第二个参数 cmd-sel) ，得到sel-imp的下标index（即搜索下标） 存入p12（cache insert写入时的哈希下标计算是 通过 sel & mask，读取时也需要通过这种方式）  
    and p12, p1, p11, LSR #48       // x12 = _cmd & mask   

//--- 非64位真机  
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4   
    and p10, p11, #~0xf         // p10 = buckets  
    and p11, p11, #0xf          // p11 = maskShift  
    mov p12, #0xffff  
    lsr p11, p12, p11               // p11 = mask = 0xffff >> p11  
    and p12, p1, p11                // x12 = _cmd & mask  
#else  
#error Unsupported cache mask storage for ARM64.  
#endif  

//--- p12是下标 p10是buckets数组首地址，下标 * 1<<4(即16) 得到实际内存的偏移量，通过buckets的首地址偏移，获取bucket存入p12寄存器  
//--- LSL #(1+PTRSHIFT)-- 实际含义就是得到一个bucket占用的内存大小 -- 相当于mask = occupied -1-- _cmd & mask -- 取余数  
    add p12, p10, p12, LSL #(1+PTRSHIFT)     
                     // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT)) -- PTRSHIFT是3  
                     
//--- 从x12（即p12）中取出 bucket 分别将imp和sel 存入 p17（存储imp） 和 p9（存储sel）
    ldp p17, p9, [x12]      // {imp, sel} = *bucket   
    
//--- 比较 sel 与 p1（传入的参数cmd）  
1:  cmp p9, p1          // if (bucket->sel != _cmd)   
//--- 如果不相等，即没有找到，请跳转至 2f  
    b.ne    2f          //     scan more   
//--- 如果相等 即cacheHit 缓存命中，直接返回imp  
    CacheHit $0         // call or return imp   
    
2:  // not hit: p12 = not-hit bucket  
//--- 如果一直都找不到， 因为是normal ，跳转至__objc_msgSend_uncached  
    CheckMiss $0            // miss if bucket->sel == 0   
//--- 判断p12（下标对应的bucket） 是否 等于 p10（buckets数组第一个元素，），如果等于，则跳转至第3步  
    cmp p12, p10        // wrap if bucket == buckets   
//--- 定位到最后一个元素（即第一个bucket）  
    b.eq    3f   
//--- 从x12（即p12 buckets首地址）- 实际需要平移的内存大小BUCKET_SIZE，得到得到第二个bucket元素，imp-sel分别存入p17-p9，即向前查找   
    ldp p17, p9, [x12, #-BUCKET_SIZE]!  // {imp, sel} = *--bucket   
//--- 跳转至第1步，继续对比 sel 与 cmd  
    b   1b          // loop   

3:  // wrap: p12 = first bucket, w11 = mask  
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16  
//--- 人为设置到最后一个元素  
//--- p11（mask）右移44位 相当于mask左移4位，直接定位到buckets的最后一个元素，缓存查找顺序是向前查找  
    add p12, p12, p11, LSR #(48 - (1+PTRSHIFT))   
                    // p12 = buckets + (mask << 1+PTRSHIFT)   
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4  
    add p12, p12, p11, LSL #(1+PTRSHIFT)  
                    // p12 = buckets + (mask << 1+PTRSHIFT)  
#else  
#error Unsupported cache mask storage for ARM64.  
#endif  

    // Clone scanning loop to miss instead of hang when cache is corrupt.  
    // The slow path may detect any corruption and halt later.  
//--- 再查找一遍缓存()  
//--- 拿到x12（即p12）bucket中的 imp-sel 分别存入 p17-p9  
    ldp p17, p9, [x12]      // {imp, sel} = *bucket   
    
//--- 比较 sel 与 p1（传入的参数cmd）  
1:  cmp p9, p1          // if (bucket->sel != _cmd)   
//--- 如果不相等，即走到第二步  
    b.ne    2f          //     scan more   
//--- 如果相等 即命中，直接返回imp  
    CacheHit $0         // call or return imp    
    
2:  // not hit: p12 = not-hit bucket  
//--- 如果一直找不到，则CheckMiss  
    CheckMiss $0            // miss if bucket->sel == 0   
//--- 判断p12（下标对应的bucket） 是否 等于 p10（buckets数组第一个元素）-- 表示前面已经没有了，但是还是没有找到
    cmp p12, p10        // wrap if bucket == buckets   
    b.eq    3f //如果等于，跳转至第3步  
//--- 从x12（即p12 buckets首地址）- 实际需要平移的内存大小BUCKET_SIZE，得到得到第二个bucket元素，imp-sel分别存入p17-p9，即向前查找  
    ldp p17, p9, [x12, #-BUCKET_SIZE]!  // {imp, sel} = *--bucket   
//--- 跳转至第1步，继续对比 sel 与 cmd  
    b   1b          // loop   

LLookupEnd$1:  
LLookupRecover$1:  
3:  // double wrap  
//--- 跳转至JumpMiss 因为是normal ，跳转至__objc_msgSend_uncached  

    JumpMiss $0   
.endmacro  

//以下是最后跳转的汇编函数  
.macro CacheHit  
.if $0 == NORMAL  
    TailCallCachedImp x17, x12, x1, x16 // authenticate and call imp  
.elseif $0 == GETIMP  
    mov p0, p17  
    cbz p0, 9f          // don't ptrauth a nil imp  
    AuthAndResignAsIMP x0, x12, x1, x16 // authenticate imp and re-sign as IMP  
9:  ret             // return IMP  
.elseif $0 == LOOKUP  
    // No nil check for ptrauth: the caller would crash anyway when they  
    // jump to a nil IMP. We don't care if that jump also fails ptrauth.  
    AuthAndResignAsIMP x17, x12, x1, x16    // authenticate imp and re-sign as IMP  
    ret             // return imp via x17  
.else  
.abort oops  
.endif  
.endmacro  

.macro CheckMiss  
    // miss if bucket->sel == 0  
.if $0 == GETIMP   
//--- 如果为GETIMP ，则跳转至 LGetImpMiss  
    cbz p9, LGetImpMiss  
.elseif $0 == NORMAL   
//--- 如果为NORMAL ，则跳转至 __objc_msgSend_uncached  
    cbz p9, __objc_msgSend_uncached  
.elseif $0 == LOOKUP   
//--- 如果为LOOKUP ，则跳转至 __objc_msgLookup_uncached  
    cbz p9, __objc_msgLookup_uncached  
.else  
.abort oops  
.endif  
.endmacro  

.macro JumpMiss  
.if $0 == GETIMP  
    b   LGetImpMiss    
.elseif $0 == NORMAL  
    b   __objc_msgSend_uncached  
.elseif $0 == LOOKUP  
    b   __objc_msgLookup_uncached  
.else  
.abort oops  
.endif  
.endmacro  
```  

最后附上伪代码流程分析(注伪代码逻辑没有那么严谨, 但是有大致的流程分析)

```
[person sayHello]  -> imp ( cache -> bucket (sel imp))  

// 获取当前的对象  
id person = 0x10000  
// 获取isa  
isa_t isa = 0x000000  
// isa -> class -> cache  
cache_t cache = isa + 16字节  

// arm64  
// mask|buckets 在一起的  
buckets = cache & 0x0000ffffffffffff  
// 获取mask  
mask = cache LSR #48  
// 下标 = mask & sel  
index = mask & p1  

// bucket 从 buckets 遍历的开始 (起始查询的bucket)  
bucket = buckets + index * 16 (sel imp = 16)  

 
int count = 0  
// CheckMiss $0  
do{  
 
    if ((bucket == buckets) && (count == 0)){ // 进入第二层判断  
        // bucket == 第一个元素  
        // bucket人为设置到最后一个元素  
        bucket = buckets + mask * 16  
        count++;  
        
    }else if (count == 1) goto CheckMiss  
        
    // {imp, sel} = *--bucket  
    // 缓存的查找的顺序是: 向前查找  
    bucket--;  
    imp = bucket.imp;  
    sel = bucket.sel;  
    
}while (bucket.sel != _cmd)  //  // bucket里面的sel 是否匹配_cmd  

// CacheHit $0  
return imp  

CheckMiss:  
    CheckMiss(normal)  
```