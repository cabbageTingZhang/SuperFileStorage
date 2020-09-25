### 1. 先来看下动态方法决议的分析

```
static NEVER_INLINE IMP  
resolveMethod_locked(id inst, SEL sel, Class cls, int behavior)  
{  
    runtimeLock.assertLocked();  
    ASSERT(cls->isRealized());  

    runtimeLock.unlock();  
    // 动态方法决议 : 给一次机会 重新查询  
    if (! cls->isMetaClass()) {  // 对象 - 类  
        // try [cls resolveInstanceMethod:sel]  
        resolveInstanceMethod(inst, sel, cls);  
    } 
    else { // 类方法 - 元类  
        // try [nonMetaClass resolveClassMethod:sel]  
        // and [cls resolveInstanceMethod:sel]  
        resolveClassMethod(inst, sel, cls);  
        if (!lookUpImpOrNil(inst, sel, cls)) {  // 为什么要有这行代码  ,因为类其实是元类的对象. 所以又执行了对象方法的查找流程
            resolveInstanceMethod(inst, sel, cls);  
        }  
    }  

    // chances are that calling the resolver have populated the cache   
    // so attempt using it  
    return lookUpImpOrForward(inst, sel, cls, behavior | LOOKUP_CACHE);  
}  
```

### 2. 通过instrumentObjcMessageSends来辅助分析

通过instrumentObjcMessageSends来分析的代码如下:  (最终打印的日志在`/private/tmp/msgSends`路径下)

```  
#import <Foundation/Foundation.h>  
#import "LGPerson.h"  

// 慢速查找   
extern void instrumentObjcMessageSends(BOOL flag);  

int main(int argc, const char * argv[]) {  
    @autoreleasepool {  

        LGPerson *person = [LGPerson alloc];  
        instrumentObjcMessageSends(YES);  
        [person sayHello];  
        instrumentObjcMessageSends(NO);  
        NSLog(@"Hello, World!");  
    }  
    return 0;  
}  
```  

我们可以看到日志的打印文件输出为:  

```  
+ LGPerson NSObject resolveInstanceMethod:  
+ LGPerson NSObject resolveInstanceMethod:  

- LGPerson NSObject forwardingTargetForSelector:  
- LGPerson NSObject forwardingTargetForSelector:  

- LGPerson NSObject methodSignatureForSelector:  
- LGPerson NSObject methodSignatureForSelector:  

- LGPerson NSObject class  

+ LGPerson NSObject resolveInstanceMethod:  
+ LGPerson NSObject resolveInstanceMethod:  

- LGPerson NSObject doesNotRecognizeSelector:  
- LGPerson NSObject doesNotRecognizeSelector:  

- LGPerson NSObject class  
- OS_xpc_serializer OS_xpc_object dealloc  
- OS_object NSObject dealloc  
+ OS_xpc_payload NSObject class  
- OS_xpc_payload OS_xpc_payload dealloc  
- NSObject NSObject dealloc  
- OS_dispatch_mach_msg OS_dispatch_object dealloc  
- OS_xpc_dictionary OS_xpc_object dealloc  
+ OS_xpc_mach_send NSObject initialize  
- OS_xpc_mach_send OS_xpc_object dealloc  
- OS_object NSObject dealloc  
- OS_object NSObject dealloc  
```

我们通过打印可以看到, 方法在找不到之后, 又依次运行了`resolveInstanceMethod ` -> `forwardingTargetForSelector ` -> `methodSignatureForSelector `

那么第一次补救, 如果是对象方法, 可以在`resolveInstanceMethod `中进行如下的举例替换

```
+ (BOOL)resolveInstanceMethod:(SEL)sel{  
    NSLog(@"%@ 来了",NSStringFromSelector(sel));  
    if (sel == @selector(say666)) {  
        NSLog(@"%@ 来了",NSStringFromSelector(sel));  

        IMP imp           = class_getMethodImplementation(self, @selector(sayMaster));  
        Method sayMMethod = class_getInstanceMethod(self, @selector(sayMaster));  
        const char *type  = method_getTypeEncoding(sayMMethod);  
        return class_addMethod(self, sel, imp, type);  
    }
    return [super resolveInstanceMethod:sel];  
}  
```

如果是类方法, 那么根据第一段代码, 是要在`resolveClassMethod`中进行拦截修正的, 举例代码如下:  

```
+ (BOOL)resolveClassMethod:(SEL)sel{  
    NSLog(@"%@ 来了",NSStringFromSelector(sel));  
    if (sel == @selector(sayNB)) {  

        IMP imp           = class_getMethodImplementation(objc_getMetaClass("LGPerson"), @selector(lgClassMethod));  
        Method sayMMethod = class_getInstanceMethod(objc_getMetaClass("LGPerson"), @selector(lgClassMethod));  
        const char *type  = method_getTypeEncoding(sayMMethod);  
        return class_addMethod(objc_getMetaClass("LGPerson"), sel, imp, type);  
    }  
    return [super resolveClassMethod:sel];  
}  
```

当然你也可以合并拦截, 不过要在基类NSObject中的分类中实现, 代码可以如下, **不过这种方法比较暴力, 不建议使用**:  

```
#import "NSObject+LG.h"  
#import <objc/message.h>  

@implementation NSObject (LG)  

// 调用方法的时候 - 分类  

+ (BOOL)resolveInstanceMethod:(SEL)sel{  
    

    NSLog(@"%@ 来了",NSStringFromSelector(sel));  
    if (sel == @selector(say666)) {  
        NSLog(@"%@ 来了",NSStringFromSelector(sel));  

        IMP imp           = class_getMethodImplementation(self, @selector(sayMaster));  
        Method sayMMethod = class_getInstanceMethod(self, @selector(sayMaster));  
        const char *type  = method_getTypeEncoding(sayMMethod);  
        return class_addMethod(self, sel, imp, type);  
    }  
    else if (sel == @selector(sayNB)) {  
        
        IMP imp           = class_getMethodImplementation(objc_getMetaClass("LGPerson"), @selector(lgClassMethod));  
        Method sayMMethod = class_getInstanceMethod(objc_getMetaClass("LGPerson"), @selector(lgClassMethod));  
        const char *type  = method_getTypeEncoding(sayMMethod);  
        return class_addMethod(objc_getMetaClass("LGPerson"), sel, imp, type);  
    }  
    return NO;  
}

/**
 
 1: 分类 - 便利
 2: 方法 - lg_model_tracffic
        - lg - model home - 奔溃 - pop Home
        - lg - mine  - mine
    切面 - SDK - 上传
 3: AOP - 封装SDK - 不处理
 4: 消息转发 - 
 
 */
```

第二次你可以在`forwardingTargetForSelector`中拦截, 也就是快速转发

```
- (id)forwardingTargetForSelector:(SEL)aSelector{  
    NSLog(@"%s - %@",__func__,NSStringFromSelector(aSelector));  
    if(aSelector == @selector(sayBad)){  
    	return [LGStudent sayHello];  
    }  
    // runtime + aSelector + addMethod + imp  
    return [super forwardingTargetForSelector:aSelector];  
}  
```

第三次你可以在`methodSignatureForSelector`进行拦截, 也就是慢速转发

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{  
    NSLog(@"%s - %@",__func__,NSStringFromSelector(aSelector));  
    return nil;  
}  

- (void)forwardInvocation:(NSInvocation *)anInvocation{  
    NSLog(@"%s - %@",__func__,anInvocation);  
    // GM  sayHello - anInvocation - 漂流瓶 - anInvocation  
    anInvocation.target = [LGStudent alloc];  
    // anInvocation 保存 - 方法  
    [anInvocation invoke];  
}  
```

最终我们可以总结成一幅图, 也就是网上比较常见的一幅图:    

![消息转发机制](https://upload-images.jianshu.io/upload_images/1367029-e787c292c7e40cf1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
本篇文章可以用一幅图来总结一下, 与上面对比[图片出处](https://www.yuque.com/u1239504/ug7g73/tg4swa):  

![文章总结](https://upload-images.jianshu.io/upload_images/1367029-5fc237c25533986d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3. 有关反汇编

在方法崩溃后, 我们可以通过指令`bt`来打印堆栈截图如下:  

![堆栈截图](https://upload-images.jianshu.io/upload_images/1367029-3309b7402570c1a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到在执行`doesNotRecognizedSelector`之前，执行`__forwarding_prep_0___`和`___forwarding___`。  
而在`__forwarding_prep_0___ `以及`___forwarding___ `输入`CoreFoundation`框架的方法, 并且不对外开源, 我们可以找到 `CoreFoundation`的可运行文件, 用反编译工具来进行反编辑就可以查看反编译猴的汇编代码来分析流程, 可以分析出跟上面我们分析的流程一下, 这里只做记录:  

![找到CoreFoundation可运行文件路径](https://upload-images.jianshu.io/upload_images/1367029-f1efc19381a1eaa9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

反编译结果(`用的Hopper Disassembler`):  

![反编译结果:](https://upload-images.jianshu.io/upload_images/1367029-e6ac55b6cc735b2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
