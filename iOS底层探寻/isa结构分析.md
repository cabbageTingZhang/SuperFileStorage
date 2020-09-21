在对象调用alloc, 之后调用的最后一个方法是`obj->initInstanceIsa`, 它的作用是将isa指针与我们的对象关联起来, 我们来分析一下isa指针.

我们这里先看下isa指针初始化的源码:  

```  
inline void   
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor)   
{   
    ASSERT(!isTaggedPointer());   
    
    if (!nonpointer) {  
        isa = isa_t((uintptr_t)cls);  
    } else {  
        ASSERT(!DisableNonpointerIsa);  
        ASSERT(!cls->instancesRequireRawIsa());  

        isa_t newisa(0);  
#if SUPPORT_INDEXED_ISA  
        ASSERT(cls->classArrayIndex() > 0);  
        newisa.bits = ISA_INDEX_MAGIC_VALUE;  
        // isa.magic is part of ISA_MAGIC_VALUE  
        // isa.nonpointer is part of ISA_MAGIC_VALUE  
        newisa.has_cxx_dtor = hasCxxDtor;  
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();  
#else  
        newisa.bits = ISA_MAGIC_VALUE;  
        // isa.magic is part of ISA_MAGIC_VALUE  
        // isa.nonpointer is part of ISA_MAGIC_VALUE  
        newisa.has_cxx_dtor = hasCxxDtor;  
        newisa.shiftcls = (uintptr_t)cls >> 3;  
#endif  
        // This write must be performed in a single store in some cases  
        // (for example when realizing a class because other threads  
        // may simultaneously try to use the class).  
        // fixme use atomics here to guarantee single-store and to  
        // guarantee memory order w.r.t. the class index table  
        // ...but not too atomic because we don't want to hurt instantiation  
        isa = newisa;  
    }
}
```  

其中通过宏`ISA_INDEX_MAGIC_VALUE ` 我们可以找到isa的宏定义(此处只写出了x86_64)(还有arm64, armv7k or arm64_32)如下:  

```  
# elif __x86_64__  
#   define ISA_MASK        0x00007ffffffffff8ULL  
#   define ISA_MAGIC_MASK  0x001f800000000001ULL  
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL  
#   define ISA_BITFIELD                                                        \  
      uintptr_t nonpointer        : 1;                                         \  
      uintptr_t has_assoc         : 1;                                         \  
      uintptr_t has_cxx_dtor      : 1;                                         \  
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ \  
      uintptr_t magic             : 6;                                         \  
      uintptr_t weakly_referenced : 1;                                         \  
      uintptr_t deallocating      : 1;                                         \  
      uintptr_t has_sidetable_rc  : 1;                                         \  
      uintptr_t extra_rc          : 8  
#   define RC_ONE   (1ULL<<56)  
#   define RC_HALF  (1ULL<<7)  
```  

我们都知道isa所占的内存大小为8字节, 一字节8位, 总共64位, 那么上面的代码的意思就是该isa指针所对应内存的顺序以及所占内存的大小. (本文在此并不做证明, 只是学习记录笔记)  

* nonpointer : 表示是否对isa指针开启指针优化 (0:纯isa指针  1:不止是类对象地址, isa中包含了类信息, 对象的引用计数等)
* has_assoc : 关联对象标志位, 0没有, 1 存在
* has_cxx_dtor :  该对象是否有C++或者Objc的析构器, 如果有析构函数, 则需要做析构逻辑, 如果没有, 则可以更快的释放对象. 
* shiftcls : 存储类指针的值, 开启指针优化的情况下, 在arm64架构中有33位用来存储类指针. 
* magic : 用于调试器判断当前对象是真的对象还是没有初始化的空间. 
* weakly_referenced : 对象是否被指向或者曾经指向一个ARC的若变量, 没有弱引用的对象可以更快释放. 
* deallocating : 标志对象是否正在释放内存. 
* has_sidetable_rc : 当对象引用计数大于10时, 则需要借用该变量存储进位. 
* extra_rc : 当表示该对象的引用计数值, 实际上是引用计数值减1, 例如, 如果对象的引用计数为10, 那么extra_rc为9. 如果引用计数大于10, 则需要使用到下面的has_sidetable_rc

#### 记录一些编译指令

*  用clang把目标文件编译成c++文件

```
clang -rewrite-objc main.m -o main.cpp
```

* UIKit报错问题  (解决方法如下)

```
clang -rewrite-objc -fobjc-arc -fobjc-runtime=ios-13.0.0 -isysroot / Applications/Xcode.app/Contents/Developer/Platforms/ iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator13.0.sdk main.m 
```

* `xcode`安装的时候顺带安装了`xcrun`命令，`xcrun`命令在`clang`的基础上进行了 一些封装，要更好用一些 

```
xcrun -sdk iphonesimulator clang -arch arm64 -rewrite-objc main.m -o main-arm64.cpp //模拟器
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main- arm64.cpp //手机
```
