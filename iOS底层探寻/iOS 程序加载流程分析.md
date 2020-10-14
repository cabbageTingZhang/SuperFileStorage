先通过一段代码抛出疑问:  

```
//ViewController.m文件 只是在.m文件中, 放出了load方法  
import "ViewController.h"  

@interface ViewController ()  

@end  

@implementation ViewController  

+ (void)load{  
    NSLog(@"%s",__func__);   
}  

- (void)viewDidLoad {  
    [super viewDidLoad];  
    // Do any additional setup after loading the view.  
}  

//在工程中的mian.m文件中, 写如下代码, 

// load -> Cxx -> main  
__attribute__((constructor)) void kcFunc(){  
    printf("来了 : %s \n",__func__);  
}  

// 内存 main() dyld image init 注册回调通知 - dyld_start  -> dyld::main()  -> main()  
// rax  
int main(int argc, char * argv[]) {  
    NSString * appDelegateClassName;  
    
    NSLog(@"1223333");  
    
    @autoreleasepool {  
        // Setup code that might create autoreleased objects goes here.  
        appDelegateClassName = NSStringFromClass([AppDelegate class]);  
    }  
    return UIApplicationMain(argc, argv, nil, appDelegateClassName);  
}  
```
在`ViewController .m`文件中的`load`方法, 以及在`main.m`文件中的`c**函数方法`以及`main`函数方法的打印顺序, 通过打印结果我们可以看到打印的顺序如下:  

```
//打印结果  
2020-09-27 15:46:32.959435+0800 002-应用程加载分析[22871:3847093] +[ViewController load]  
来了 : kcFunc   
2020-09-27 15:46:32.959926+0800 002-应用程加载分析[22871:3847093] 1223333  
```

即`load方法`  -> `C**方法` -> `main函数` 为什么执行顺序会是这样的?   
我们在每个方法中加断点并打印堆栈, 结果如下:  

```
//在load函数中的断点 bt
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
  * frame #0: 0x000000010970be17 002-应用程加载分析`+[ViewController load](self=ViewController, _cmd="load") at ViewController.m:17:5
    frame #1: 0x00007fff201805e3 libobjc.A.dylib`load_images + 1442
    frame #2: 0x000000010971fe54 dyld_sim`dyld::notifySingle(dyld_image_states, ImageLoader const*, ImageLoader::InitializerTimingList*) + 425
    frame #3: 0x000000010972e887 dyld_sim`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 437
    frame #4: 0x000000010972cbb0 dyld_sim`ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 188
    frame #5: 0x000000010972cc50 dyld_sim`ImageLoader::runInitializers(ImageLoader::LinkContext const&, ImageLoader::InitializerTimingList&) + 82
    frame #6: 0x00000001097202a9 dyld_sim`dyld::initializeMainExecutable() + 199
    frame #7: 0x0000000109724d50 dyld_sim`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 4431
    frame #8: 0x000000010971f1c7 dyld_sim`start_sim + 122
    frame #9: 0x0000000117efa85c dyld`dyld::useSimulatorDyld(int, macho_header const*, char const*, int, char const**, char const**, char const**, unsigned long*, unsigned long*) + 2308
    frame #10: 0x0000000117ef84f4 dyld`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 837
    frame #11: 0x0000000117ef3227 dyld`dyldbootstrap::start(dyld3::MachOLoaded const*, int, char const**, dyld3::MachOLoaded const*, unsigned long*) + 453
    frame #12: 0x0000000117ef3025 dyld`_dyld_start + 37
    
    //在C**函数中的断点 bt
    (lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
  * frame #0: 0x0000000110000084 002-应用程加载分析`kcFunc at main.m:13:5
    frame #1: 0x000000011002794b dyld_sim`ImageLoaderMachO::doModInitFunctions(ImageLoader::LinkContext const&) + 537
    frame #2: 0x0000000110027d34 dyld_sim`ImageLoaderMachO::doInitialization(ImageLoader::LinkContext const&) + 40
    frame #3: 0x0000000110022899 dyld_sim`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 455
    frame #4: 0x0000000110020bb0 dyld_sim`ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 188
    frame #5: 0x0000000110020c50 dyld_sim`ImageLoader::runInitializers(ImageLoader::LinkContext const&, ImageLoader::InitializerTimingList&) + 82
    frame #6: 0x00000001100142a9 dyld_sim`dyld::initializeMainExecutable() + 199
    frame #7: 0x0000000110018d50 dyld_sim`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 4431
    frame #8: 0x00000001100131c7 dyld_sim`start_sim + 122
    frame #9: 0x00000001165a985c dyld`dyld::useSimulatorDyld(int, macho_header const*, char const*, int, char const**, char const**, char const**, unsigned long*, unsigned long*) + 2308
    frame #10: 0x00000001165a74f4 dyld`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 837
    frame #11: 0x00000001165a2227 dyld`dyldbootstrap::start(dyld3::MachOLoaded const*, int, char const**, dyld3::MachOLoaded const*, unsigned long*) + 453
    frame #12: 0x00000001165a2025 dyld`_dyld_start + 37

```

我们可以通过分析`dyld`的源码来分析程序的加载流程, 用一副图可以总结如下:  

![dyld流程分析图.png](https://upload-images.jianshu.io/upload_images/1367029-6b7e0ac564ebda45.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

我们可以看到在程序的加载过程中, `dyld`起到了至关重要的作用.  

#### dyld简介

`dyld（the dynamic link editor）`是苹果的动态链接器，是苹果操作系统一个重要组成部分，在系统内核做好程序准备工作之后，交由dyld负责余下的工作。而且它是开源的，任何人可以通过苹果官网下载它的源码来阅读理解它的运作方式，了解系统加载动态库的细节。[dyld下载地址](http://opensource.apple.com/tarballs/dyld)  

在iOS系统中，几乎所有的程序都会用到动态库，而动态库在加载的时候都需要用`dyld`（位于`/usr/lib/dyld`）程序进行链接。很多系统库几乎都是每个程序都要用到的，与其在每个程序运行的时候一个一个将这些动态库都加载进来，还不如先把它们打包好，一次加载进来来的快。 

#### dyld共享缓存

 dyld加载时，为了优化程序启动，启用了共享缓存（shared cache）技术。共享缓存会在进程启动时被dyld映射到内存中，之后，当任何Mach-O映像加载时，dyld首先会检查该Mach-O映像与所需的动态库是否在共享缓存中，如果存在，则直接将它在共享内存中的内存地址映射到进程的内存地址空间。在程序依赖的系统动态库很多的情况下，这种做法对程序启动性能是有明显提升的。
 
#### dyld程序启动过程  

 系统先读取App的可执行文件（Mach-O文件），从里面获得dyld的路径，然后加载dyld，dyld去初始化运行环境，开启缓存策略，加载程序相关依赖库(其中也包含我们的可执行文件)，并对这些库进行链接，最后调用每个依赖库的初始化方法，在这一步，runtime被初始化。当所有依赖库的初始化后，轮到最后一位(程序可执行文件)进行初始化，在这时runtime会对项目中所有类进行类结构初始化，然后调用所有的load方法。最后dyld返回main函数地址，main函数被调用，我们便来到了熟悉的程序入口。 

* ImageLoader : 用于辅助加载特定可执行文件格式的类，程序中对应实例可简称为image(如程序可执行文件，Framework库，bundle文件)。  

####  _objc_init符号断点:  
 
![1419656-0cb5ee83eefa4ac6.png](https://upload-images.jianshu.io/upload_images/1367029-4069fb7f8e27e57f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里可以看到_objc_init调用的顺序，先libSystem_initializer调用libdispatch_init再到_objc_init初始化runtime。

runtime初始化后不会闲着，在_objc_init中注册了几个通知，从dyld这里接手了几个活，其中包括负责初始化相应依赖库里的类结构，调用依赖库里所有的load方法。

就拿sMainExcuatable来说，它的initializer方法是最后调用的，当initializer方法被调用前dyld会通知runtime进行类结构初始化，然后再通知调用load方法，这些目前还发生在main函数前，但由于lazy bind机制，依赖库多数都是在使用时才进行bind，所以这些依赖库的类结构初始化都是发生在程序里第一次使用到该依赖库时才进行的。


文章参考: [【IOS开发高级系列】dyld专题](https://www.jianshu.com/p/5f337da8fbef)