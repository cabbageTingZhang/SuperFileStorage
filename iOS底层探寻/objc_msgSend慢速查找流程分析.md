有关[上篇文章](https://www.jianshu.com/p/a485568bf033)分析了objc_msgSend快速查找流程, 当在类的方法缓存中查找不到的话, objc_msgSend就会进入慢速查找流程, 我们来分析一下:  (注, 以下代码都是在arm64环境下的)

### objc_msgSend慢速查找流程
#### 1. CheckMiss还是JumpMiss方法实现
在快速查找流程中, 如果没有找到方法实现, 无论是走到CheckMiss还是JumpMiss, 最终都会走到__objc_msgSend_uncached汇编函数.  

```   
//有关CheckMiss 以及 JumpMiss代码, 以为没有找到, 所以传进来的参数都为NORMAL, 转而走__objc_msgSend_uncached
.macro CheckMiss  
	// miss if bucket->sel == 0  
.if $0 == GETIMP  
	cbz	p9, LGetImpMiss  
.elseif $0 == NORMAL  
	cbz	p9, __objc_msgSend_uncached  
.elseif $0 == LOOKUP  
	cbz	p9, __objc_msgLookup_uncached  
.else  
.abort oops  
.endif  
.endmacro  

.macro JumpMiss  
.if $0 == GETIMP  
	b	LGetImpMiss  
.elseif $0 == NORMAL  
	b	__objc_msgSend_uncached  
.elseif $0 == LOOKUP  
	b	__objc_msgLookup_uncached  
.else  
.abort oops  
.endif  
.endmacro  
```  

#### 2. __objc_msgSend_uncached汇编实现

```
	//其中最重要的方法是MethodTableLookup, 即查询方法列表  
	STATIC_ENTRY __objc_msgSend_uncached  
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves  

	// THIS IS NOT A CALLABLE C FUNCTION  
	// Out-of-band p16 is the class to search  
	
	MethodTableLookup  
	TailCallFunctionPointer x17  

	END_ENTRY __objc_msgSend_uncached  
```
#### 3. 注意点
在MethodTableLookup的汇编实现中, 我们可以看到最重要的是_lookUpImpOrForward的方法, 然后全局搜索_lookUpImpOrForward发现搜不到实现方法, 说明该方法并不是汇编实现的, 需要去C/C++方法中查找

* c/c++中调动汇编, 去查找汇编时, 需要将需要搜索的方法多加一个下划线.  
* 汇编中调用c/c++方法, 去查找c/c++方法时, 需要将需要查找的方法去掉一个下划线.  

####4. lookUpImpOrForward实现
全局搜索lookUpImpOrForward方法, 在objc-runtime-new.mm中可以找到实现代码, 如下:  

```
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)  
{  
    // 定义的消息转发  
    const IMP forward_imp = (IMP)_objc_msgForward_impcache;   
    IMP imp = nil;  
    Class curClass;  

    runtimeLock.assertUnlocked();  

    // 快速查找，如果找到则直接返回imp  
    //目的：防止多线程操作时，刚好调用函数，此时缓存进来了  
    if (fastpath(behavior & LOOKUP_CACHE)) {   
        imp = cache_getImp(cls, sel);  
        if (imp) goto done_nolock;  
    }  
    
    //加锁，目的是保证读取的线程安全  
    runtimeLock.lock();  
    
    //判断是否是一个已知的类：判断当前类是否是已经被认可的类，即已经加载的类  
    checkIsKnownClass(cls);    
    
    //判断类是否实现，如果没有，需要先实现，此时的目的是为了确定父类链，方法后续的循环  
    if (slowpath(!cls->isRealized())) {   
        cls = realizeClassMaybeSwiftAndLeaveLocked(cls, runtimeLock);  
    }  

    //判断类是否初始化，如果没有，需要先初始化  
    if (slowpath((behavior & LOOKUP_INITIALIZE) && !cls->isInitialized())) {   
        cls = initializeAndLeaveLocked(cls, inst, runtimeLock);  
    }  

    runtimeLock.assertLocked();  
    curClass = cls;  

    //----查找类的缓存  
    
    // unreasonableClassCount -- 表示类的迭代的上限  
    //（猜测这里递归的原因是attempts在第一次循环时作了减一操作，然后再次循环时,仍在上限的范围内，所以可以继续递归）  
    for (unsigned attempts = unreasonableClassCount();;) {   
        //---当前类方法列表（采用二分查找算法），如果找到，则返回，将方法缓存到cache中  
        Method meth = getMethodNoSuper_nolock(curClass, sel);  
        if (meth) {  
            imp = meth->imp;  
            goto done;  
        }  
        //当前类 = 当前类的父类，并判断父类是否为nil   
        if (slowpath((curClass = curClass->superclass) == nil)) {  
            //--未找到方法实现，方法解析器也不行，使用转发  
            imp = forward_imp;  
            break;  
        }  
 
        // 如果父类链中存在循环，则停止  
        if (slowpath(--attempts == 0)) {  
            _objc_fatal("Memory corruption in class list.");  
        }  

        // --父类缓存   
        // Superclass cache.  
        // 缓存 - lookupimp - 慢速 -  
        // 自己验证 当前类的 缓存 teacher - person - NSObject - nil  并非无限循环 !
        imp = cache_getImp(curClass, sel);  
        if (slowpath(imp == forward_imp)) {   
            // 如果在父类中找到了forward，则停止查找，且不缓存，首先调用此类的方法解析器  
            break;  
        }  
        if (fastpath(imp)) {  
            //如果在父类中，找到了此方法，将其存储到cache中  
            goto done;  
        }    
    }  

    //没有找到方法实现，尝试一次方法解析  

    if (slowpath(behavior & LOOKUP_RESOLVER)) {  
        //动态方法决议的控制条件，表示流程只走一次  
        behavior ^= LOOKUP_RESOLVER;   
        return resolveMethod_locked(inst, sel, cls, behavior);  
    }  

 done:  
    //存储到缓存   (为下次直接从缓存里面快速查找做准备)
    log_and_fill_cache(cls, imp, sel, inst, curClass);   
    //解锁  
    runtimeLock.unlock();  
 done_nolock:  
    if (slowpath((behavior & LOOKUP_NIL) && imp == forward_imp)) {  
        return nil;  
    }  
    return imp;  
}
```

####5. 分析getMethodNoSuper_nolock

```
//其实就是查找类的方法链表methed_list
getMethodNoSuper_nolock(Class cls, SEL sel)  
{
    runtimeLock.assertLocked();  

    ASSERT(cls->isRealized());  
    // fixme nil cls?   
    // fixme nil sel?  

    auto const methods = cls->data()->methods();  
    for (auto mlists = methods.beginLists(),  
              end = methods.endLists();  
         mlists != end;  
         ++mlists)  
    {
        // <rdar://problem/46904873> getMethodNoSuper_nolock is the hottest  
        // caller of search_method_list, inlining it turns  
        // getMethodNoSuper_nolock into a frame-less function and eliminates  
        // any store from this codepath.  
        method_t *m = search_method_list_inline(*mlists, sel);  
        if (m) return m;  
    }  

    return nil;  
}  

//search_method_list_inline的实现
ALWAYS_INLINE static method_t *  
search_method_list_inline(const method_list_t *mlist, SEL sel)  
{  
    int methodListIsFixedUp = mlist->isFixedUp();  
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);  
    
    if (fastpath(methodListIsFixedUp && methodListHasExpectedSize)) {  
        return findMethodInSortedMethodList(sel, mlist);  
    } else {  
        // Linear search of unsorted method list  
        for (auto& meth : *mlist) {  
            if (meth.name == sel) return &meth;  
        }  
    }  

#if DEBUG  
    // sanity-check negative results  
    if (mlist->isFixedUp()) {  
        for (auto& meth : *mlist) {  
            if (meth.name == sel) {  
                _objc_fatal("linear search worked when binary search did not");  
            }  
        }  
    }  
#endif  

    return nil;  
}  
```
####6. findMethodInSortedMethodList中用的是二分查找
**二分查找也称折半查找（Binary Search），它是一种效率较高的查找方法。但是，折半查找要求线性表必须采用顺序存储结构，而且表中元素按关键字有序排列。**  

```
ALWAYS_INLINE static method_t *  
findMethodInSortedMethodList(SEL key, const method_list_t *list)  
{  
    ASSERT(list);  

    const method_t * const first = &list->first;  
    const method_t *base = first;  
    const method_t *probe;  
    uintptr_t keyValue = (uintptr_t)key; //key 等于 say666  
    uint32_t count;  
    //base相当于low，count是max，probe是middle，这就是二分  
    for (count = list->count; count != 0; count >>= 1) {  
        //从首地址+下标 --> 移动到中间位置（count >> 1 左移1位即 count/2 = 4）  
        probe = base + (count >> 1);   
        
        uintptr_t probeValue = (uintptr_t)probe->name;  
        
        //如果查找的key的keyvalue等于中间位置（probe）的probeValue，则直接返回中间位置  
        if (keyValue == probeValue) {   
            // -- while 平移 -- 排除分类重名方法  
            while (probe > first && keyValue == (uintptr_t)probe[-1].name) {  
                //排除分类重名方法（方法的存储是先存储类方法，在存储分类---按照先进后出的原则，分类方法最先出，而我们要取的类方法，所以需要先排除分类方法）  
                //如果是两个分类，就看谁先进行加载  
                probe--;  
            }  
            return (method_t *)probe;  
        }  
        
        //如果keyValue 大于 probeValue，就往probe即中间位置的右边查找  
        if (keyValue > probeValue) {   
            base = probe + 1;  
            count--;  
        }  
    }  
    
    return nil;  
}  
```

### 消息转发流程

####1. forward_imp 
`const IMP forward_imp = (IMP)_objc_msgForward_impcache;`
####2. _objc_msgForward_impcache 汇编实现: 

```
//__objc_msgForward_impcache调用__objc_msgForward
STATIC_ENTRY __objc_msgForward_impcache  

// No stret specialization.  
b   __objc_msgForward  

END_ENTRY __objc_msgForward_impcache  

//__objc_msgForward调用TailCallFunctionPointer x17
ENTRY __objc_msgForward  

adrp    x17, __objc_forward_handler@PAGE  
ldr p17, [x17, __objc_forward_handler@PAGEOFF]  
TailCallFunctionPointer x17  

END_ENTRY __objc_msgForward  
```

####3. TailCallFunctionPointer 
TailCallFunctionPointer 就是返回指针的值, 返回x17, x17的值是__objc_forward_handler方法确定的

```
.macro TailCallFunctionPointer  
    // $0 = function pointer value  
    braaz   $0  
.endmacro  
```

####4. __objc_forward_handler

```
objc_defaultForwardHandler(id self, SEL sel)  
{  
    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "  
                "(no message forward handler is installed)",   
                class_isMetaClass(object_getClass(self)) ? '+' : '-',   
                object_getClassName(self), sel_getName(sel), self);  
}  
void *_objc_forward_handler = (void*)objc_defaultForwardHandler;  
```
