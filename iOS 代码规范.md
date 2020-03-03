#iOS 代码规范

### <a name="目录"></a>目录
1. [有关命名](#有关命名)
2. [Enum](#Enum)
3. [属性的注释](#属性的注释)
4. [方法的注释](#方法的注释)
5. [Delegate](#Delegate)
6. [代码齐整](#代码齐整)
7. [命名头文件](#命名头文件)
8. [头文件引入](#头文件引入)
9. [TODO](#TODO)
10. [对method进行分组](#对method进行分组)
11. [if语句](#if语句)
12. [Switch语句](#Switch语句)
13. [协议（Protocols)](#Protocols)
14. [闭包（Blocks)](#Blocks)
15. [数据结构的语法糖](#数据结构的语法糖)
16. [三元运算符](#三元运算符)

文档的目的是描述一些关于iOS编程的标准和惯例，并指导编写稳定的iOS代码。这些都是基于健全的、充分证明过的软件工程基础上的，并使得代码 容易理解、维护和增。更重要的是，遵循这些规则，iOS开发人员的编程效率将会显著的得到提高。因为不必花费时间来重新编写代码，而是更容易的在开发过程 中，对代码进行修改。最后，遵循这些规则，可以保持代码的连续性和粘性，提升开发组的效率。

#### <a name="有关命名"></a>有关命名 [↩](#目录)

函数、属性名、全局变量、局部变量、局部常量统一使用驼峰命名规范。

* 属性(property)编写规范：头字母小写、驼峰命名方式，属性名需标明是否原子性和修饰词。

	格式：@property“空格”(nonatomic/natomic,“空格”strong/weak/retain/copy/assign)“空格”NSString 

```Objective-C
@property(nonatomic, strong) POWebProgressLayer *webProgresser;
@property(nonatomic, copy) void(^setTitleBlock)(NSString *titleStr);
@property(nonatomic, assign)BOOL hasFlags;
```

* 函数(方法)编写规范：头字母小写、驼峰命名方式，其中参数命名规范也一样。

	格式：“-/+”“空格”“返回值类型”“函数名（option：参数）”

```Objective-C
- (void)openVideoStream;
+ (NSString *)getChatIdWithPhoneNumber:(NSString *)number;
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(NSString*)defaultText initiatedByFrame:(WKFrameInfo *)frame;

```

* 方法命名
如果方法表示让对象执行一个动作，使用动词打头来命名，注意不要使用do，does这种多余的关键字，动词本身的暗示就足够了：

```Objective-C
//动词打头的方法表示让对象执行一个动作
- (void)invokeWithTarget:(id)target;
- (void)selectTabViewItem:(NSTabViewItem *)tabViewItem;
```

如果方法是为了获取对象的一个属性值，直接用属性名称来命名这个方法，注意不要添加get或者其他的动词前缀：

```Objective-C
//正确，使用属性名来命名方法
- (NSSize)cellSize;

//错误，添加了多余的动词前缀
- (NSSize)calcCellSize;
- (NSSize)getCellSize;
```

对于有多个参数的方法，务必在每一个参数前都添加关键词，关键词应当清晰说明参数的作用：

```Objective-C
//正确，保证每个参数都有关键词修饰
- (void)sendAction:(SEL)aSelector toObject:(id)anObject forAllCells:(BOOL)flag;

//错误，遗漏关键词
- (void)sendAction:(SEL)aSelector :(id)anObject :(BOOL)flag;

//正确
- (id)viewWithTag:(NSInteger)aTag;

//错误，关键词的作用不清晰
- (id)taggedView:(int)aTag;
```

不要用and来连接两个参数，通常and用来表示方法执行了两个相对独立的操作（从设计上来说，这时候应该拆分成两个独立的方法）：

```Objective-C
//错误，不要使用"and"来连接参数
- (int)runModalForDirectory:(NSString *)path andFile:(NSString *)name andTypes:(NSArray *)fileTypes;

//正确，使用"and"来表示两个相对独立的操作
- (BOOL)openFile:(NSString *)fullPath withApplication:(NSString *)appName andDeactivate:(BOOL)flag;
```

* 宏和常量命名
  对于宏定义的常量
  
  格式：#define 预处理定义的常量全部大写，单词间用 _ 分隔

  宏定义中如果包含表达式或变量，表达式或变量必须用小括号括起来。
  
  对于类型常量
  
  对于局限于某编译单元(实现文件)的常量，以字符k开头，例如kAnimationDuration，且需要以static const修饰
  
  对于定义于类头文件的常量，外部可见，则以定义该常量所在类的类名开头，例如EOCViewClassAnimationDuration, 仿照苹果风格，在头文件中进行extern声明，在实现文件中定义其值
  
```Objective-C
//宏定义的常量 
#define ANIMATION_DURATION    0.3 
#define MY_MIN(A, B)  ((A)>(B)?(B):(A)) 

//局部类型常量 
static const NSTimeInterval kAnimationDuration = 0.3; 

//外部可见类型常量 
//EOCViewClass.h 
extern const NSTimeInterval EOCViewClassAnimationDuration; 
extern NSString *const EOCViewClassStringConstant;  //字符串类型 

//EOCViewClass.m 
const NSTimeInterval EOCViewClassAnimationDuration = 0.3; 
NSString *const EOCViewClassStringConstant = @"EOCStringConstant";  
```

#### <a name="Enum"></a>Enum	[↩](#目录)
Enum类型的命名与类的命名规则一致,Enum中枚举内容的命名需要以该Enum类型名称开头,NS_ENUM定义通用枚举，NS_OPTIONS定义位移枚举

```
typedef NS_ENUM(NSInteger, UIViewAnimationTransition) { 
    UIViewAnimationTransitionNone, 
    UIViewAnimationTransitionFlipFromLeft, 
    UIViewAnimationTransitionFlipFromRight, 
    UIViewAnimationTransitionCurlUp, 
    UIViewAnimationTransitionCurlDown, 
}; 

typedef NS_OPTIONS(NSUInteger, UIControlState) { 
    UIControlStateNormal       = 0, 
    UIControlStateHighlighted  = 1  
};
```

#### <a name="属性的注释"></a>属性的注释  [↩](#目录)
```Objective-C
/** 家长端发布：宝宝名称 */
@property (nonatomic, copy) NSString *babyNameNew;
```

#### <a name="方法的注释"></a>方法的注释 [↩](#目录)

```Objective-C
公用方法的注释:
/**
 班级家长的APP使用详情

 @param classId 班级id
 @param useType 使用情况(0表示没有使用过,1表示使用过)
 @param block 请求成功返回的数据以及是否断网
 */
+ (void)appUseConditionClzParentDetailWithClassId:(NSString *)classId UseType:(NSString *)useType completionBlock:(void (^)(BOOL isWifi, LBStatiPanBase *statisticsBase))block;

私有方法的注释:
[self getHttpDataSomething];//为发布准备数据

- (void)getHttpDataSomething{//准备数据请求的数据

}
```

#### <a name="Delegate"></a>Delegate   [↩](#目录)
用delegate做后缀，如
用optional修饰可以不实现的方法，用required修饰必须实现的方法
当你的委托的方法过多, 可以拆分数据部分和其他逻辑部分, 数据部分用dataSource做后缀. 如
使用did和will通知Delegate已经发生的变化或将要发生的变化。
类的实例必须为回调方法的参数之一
回调方法的参数只有类自己的情况，方法名要符合实际含义
回调方法存在两个以上参数的情况，以类的名字开头，以表明此方法是属于哪个类的

```Objective-C
@protocol UITableViewDataSource    
@required   
 
//回调方法存在两个以上参数 
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section;   
 
@optional 
  
//回调方法的参数只有类自己 
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView;// Default is 1 if not implemented 

@protocol UITableViewDelegate    
 
@optional 
  
 
//使用`did`和`will`通知`Delegate` 
- (nullable NSIndexPath *)tableView:(UITableView *)tableView willSelectRowAtIndexPath:(NSIndexPath *)indexPath; 
 
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath;  
```

#### <a name="代码齐整"></a>注意代码对齐,整洁,规范 [↩](#目录)

```Objective-C
方法内代码之间尽量少空格
    - (void)viewDidLoad {
        [super viewDidLoad];
        [self getHttpDataSomething];//为发布准备数据
        [self isTeacherNeedDoingSomething];//教师端需要做的事情
    }
    
方法与方法之间隔一行
方法体之间的括号对齐方式:
- (void)didBtnColorChange:(UIButton *)sender{//发布按钮点击颜色变深
    sender.backgroundColor = [UIColor colorWithHexString:@"#48c881"];
}

- (void)getHttpDataSomething{//准备数据请求的数据

}
```

#### <a name="命名头文件"></a> 命名头文件  [↩](#目录)
源码的头文件名应该清晰地暗示它的功能和包含的内容：

1. 如果头文件内只定义了单个类或者协议，直接用类名或者协议名来命名头文件，比如NSLocale.h定义了NSLocale类。
2. 如果头文件内定义了一系列的类、协议、类别，使用其中最主要的类名来命名头文件，比如NSString.h定义了NSString和NSMutableString。
3. 每一个Framework都应该有一个和框架同名的头文件，包含了框架中所有公共类头文件的引用，比如Foundation.h
Framework中有时候会实现在别的框架中类的类别扩展，这样的文件通常使用被扩展的框架名+Additions的方式来命名，比如NSBundleAdditions.h。

#### <a name="头文件引入"></a>头文件引入(引入不必要的文件, 会增加编译时间) [↩](#目录)

	1. 除非确有必要, 否则不要引入头文件. 一般来说, 应在某个类的头文件中使用向前声明来提及别的类, 并在实现文件中引入那些类的头文件. 这样做可以尽量降低类之间的耦合.
	2. 有时无法使用向前声明, 比如要声明某个类遵循一项协议. 这种情况下,  尽量把"该类遵循某协议" 的这条声明移至"calss-continuation"分类中. 如果不行的话, 就把协议单独放在一个头文件中, 然后将其引入. 

#### <a name="TODO"></a> TODO  [↩](#目录)
使用//TODO:说明 标记一些未完成的或完成的不尽如人意的地方

```Objective-C
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions { 
    //TODO:增加初始化 
    return YES; 
}
```

#### <a name="对method进行分组"></a> 对method进行分组   [↩](#目录)
使用#pragma mark -对method进行分组

```Objective-C
#pragma mark - Life Cycle Methods 
- (instancetype)init 
- (void)dealloc   
- (void)viewWillAppear:(BOOL)animated 
- (void)viewDidAppear:(BOOL)animated 
- (void)viewWillDisappear:(BOOL)animated 
- (void)viewDidDisappear:(BOOL)animated   
 
#pragma mark - Override Methods 
#pragma mark - Intial Methods   
#pragma mark - Network Methods   
#pragma mark - Target Methods  
#pragma mark - Public Methods   
#pragma mark - Private Methods   
#pragma mark - UITableViewDataSource   
#pragma mark - UITableViewDelegate    
#pragma mark - Lazy Loads  
#pragma mark - NSCopying    
#pragma mark - NSObject  Methods  
```

#### <a name="if语句"></a> if语句  [↩](#目录)
* 须列出所有分支(穷举所有的情况)，而且每个分支都须给出明确的结果。

```Objective-C
//推荐这样写
var hintStr; 
if (count < 3) { 
  hintStr = "Good"; 
} 
else { 
  hintStr = ""; 
}

//不推荐这样写
var hintStr; 
if (count < 3) { 
 hintStr = "Good"; 
} 
```

* 不要使用过多的分支，要善于使用return来提前返回错误的情况，把最正确的情况放到最后返回

```Objective-C
//推荐这样写
if (!user.UserName) return NO; 
if (!user.Password) return NO; 
if (!user.Email) return NO;   
return YES; 

//不推荐这样写
BOOL isValid = NO; 
if (user.UserName) { 
    if (user.Password) { 
        if (user.Email) isValid = YES; 
    } 
} 
return isValid; 
```

* 条件过多，过长的时候应该换行。条件表达式如果很长，则需要将他们提取出来赋给一个BOOL值，或者抽取出一个方法

```Objective-C
//推荐这样写
if (condition1 && 
    condition2 && 
    condition3 && 
    condition4) { 
  // Do something 
}  

BOOL finalCondition = condition1 && condition2 && condition3 && condition4 
if (finalCondition) { 
  // Do something 
} 
 
if ([self canDelete]){ 
  // Do something 
} 
 
- (BOOL)canDelete { 
    BOOL finalCondition1 = condition1 && condition2 
    BOOL finalCondition2 =  condition3 && condition4 
    return condition1 && condition2; 
}  

//不推荐这样写
if (condition1 && condition2 && condition3 && condition4) { 
  // Do something 
}  
```

* 条件语句的判断应该是变量在右，常量在左。

```Objective-C
//推荐这样写
if (6 == count) { 
 
}   

if (!object) { 
 
}  
//不推荐这样写
if (count == 6) { 
 
} 
 
if (object == nil) { 
 
}  
//if (object == nil)容易误写成赋值语句,if (!object)写法很简洁
```

* 每个分支的实现代码都须被大括号包围

```Objective-C
//推荐这样写
if (!error) { 
  return success; 
}  

//不推荐这样写
if (!error) 
  return success;  
  
//也可以这么写
  if (!error) return success; 
```

#### <a name="Switch语句"></a> Switch语句  [↩](#目录)
* 每个分支都必须用大括号括起来

```Objective-C
//推荐这样写
switch (integer) {    
  case 1:  {  
    // ...    
   }  
    break;    
  case 2: {   
    // ...    
    break;  
  }    
  default:{  
    // ...    
    break;  
  }  
}  
```

* 使用枚举类型时，不能有default分支， 除了使用枚举类型以外，都必须有default分支

```Objective-C
RWTLeftMenuTopItemType menuType = RWTLeftMenuTopItemMain;    
switch (menuType) {    
  case RWTLeftMenuTopItemMain: {  
    // ...    
    break;  
   }  
  case RWTLeftMenuTopItemShows: {  
    // ...    
    break;  
  }  
  case RWTLeftMenuTopItemSchedule: {  
    // ...    
    break;  
  }  
} 
//在Switch语句使用枚举类型的时候，如果使用了default分支，在将来就无法通过编译器来检查新增的枚举类型了。
```

#### <a name="Protocols"></a> 协议（Protocols）  [↩](#目录)
在书写协议的时候注意用<>括起来的协议和类型名之间是没有空格的，比如IPCConnectHandler()<IPCPreconnectorDelegate>,这个规则适用所有书写协议的地方，包括函数声明、类声明、实例变量等等：

```Objective-C
@interface MyProtocoledClass : NSObject<NSWindowDelegate> {
 @private
    id<MyFancyDelegate> _delegate;
}

- (void)setDelegate:(id<MyFancyDelegate>)aDelegate;
@end
```

#### <a name="Blocks"></a> 闭包（Blocks）  [↩](#目录)
根据block的长度，有不同的书写规则：

较短的block可以写在一行内。
如果分行显示的话，block的右括号}应该和调用block那行代码的第一个非空字符对齐。
block内的代码采用4个空格的缩进。
如果block过于庞大，应该单独声明成一个变量来使用。
^和(之间，^和{之间都没有空格，参数列表的右括号)和{之间有一个空格。

```Objective-C
//较短的block写在一行内
[operation setCompletionBlock:^{ [self onOperationDone]; }];

//分行书写的block，内部使用4空格缩进
[operation setCompletionBlock:^{
    [self.delegate newDataAvailable];
}];

//使用C语言API调用的block遵循同样的书写规则
dispatch_async(_fileIOQueue, ^{
    NSString* path = [self sessionFilePath];
    if (path) {
      // ...
    }
});

//较长的block关键字可以缩进后在新行书写，注意block的右括号'}'和调用block那行代码的第一个非空字符对齐
[[SessionService sharedService]
    loadWindowWithCompletionBlock:^(SessionWindow *window) {
        if (window) {
          [self windowDidLoad:window];
        } else {
          [self errorLoadingWindow];
        }
    }];

//较长的block参数列表同样可以缩进后在新行书写
[[SessionService sharedService]
    loadWindowWithCompletionBlock:
        ^(SessionWindow *window) {
            if (window) {
              [self windowDidLoad:window];
            } else {
              [self errorLoadingWindow];
            }
        }];

//庞大的block应该单独定义成变量使用
void (^largeBlock)(void) = ^{
    // ...
};
[_operationQueue addOperationWithBlock:largeBlock];

//在一个调用中使用多个block，注意到他们不是像函数那样通过':'对齐的，而是同时进行了4个空格的缩进
[myObject doSomethingWith:arg1
    firstBlock:^(Foo *a) {
        // ...
    }
    secondBlock:^(Bar *b) {
        // ...
    }];
```

* Block循环引用问题

当使用代码块和异步分发的时候，要注意避免引用循环。 总是使用 weak 来引用对象，避免引用循环。（译者注：这里更为优雅的方式是采用影子变量@weakify/@strongify 这里有更为详细的说明） 此外，把持有 block 的属性设置为 nil (比如 self.completionBlock = nil) 是一个好的实践。它会打破 block 捕获的作用域带来的引用循环。

```Objective-C
__weak __typeof(self) weakSelf = self;
[self executeBlock:^(NSData *data, NSError *error) {
    [weakSelf doSomethingWithData:data];
}];

//不要这样写
[self executeBlock:^(NSData *data, NSError *error) {
    [self doSomethingWithData:data];
}];

//多个语句的例子
__weak __typeof(self)weakSelf = self;
[self executeBlock:^(NSData *data, NSError *error) {
    __strong __typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf) {
        [strongSelf doSomethingWithData:data];
        [strongSelf doSomethingWithData:data];
    }
}]

//不要这样写
__weak __typeof(self)weakSelf = self;
[self executeBlock:^(NSData *data, NSError *error) {
    [weakSelf doSomethingWithData:data];
    [weakSelf doSomethingWithData:data];
}];
```

#### <a name="数据结构的语法糖"></a> 数据结构的语法糖  [↩](#目录)
应该使用可读性更好的语法糖来构造NSArray，NSDictionary等数据结构，避免使用冗长的alloc,init方法。

如果构造代码写在一行，需要在括号两端留有一个空格，使得被构造的元素于与构造语法区分开来：

```Objective-C
//正确，在语法糖的"[]"或者"{}"两端留有空格
NSArray *array = @[ [foo description], @"Another String", [bar description] ];
NSDictionary *dict = @{ NSForegroundColorAttributeName : [NSColor redColor] };

//不正确，不留有空格降低了可读性
NSArray* array = @[[foo description], [bar description]];
NSDictionary* dict = @{NSForegroundColorAttributeName: [NSColor redColor]};
```

如果构造代码不写在一行内，构造元素需要使用两个空格来进行缩进，右括号]或者}写在新的一行，并且与调用语法糖那行代码的第一个非空字符对齐：

```Objective-C
NSArray *array = @[
  @"This",
  @"is",
  @"an",
  @"array"
];

NSDictionary *dictionary = @{
  NSFontAttributeName : [NSFont fontWithName:@"Helvetica-Bold" size:12],
  NSForegroundColorAttributeName : fontColor
};
```

构造字典时，字典的Key和Value与中间的冒号:都要留有一个空格，多行书写时，也可以将Value对齐：

```Objective-C
//正确，冒号':'前后留有一个空格
NSDictionary *option1 = @{
  NSFontAttributeName : [NSFont fontWithName:@"Helvetica-Bold" size:12],
  NSForegroundColorAttributeName : fontColor
};

//正确，按照Value来对齐
NSDictionary *option2 = @{
  NSFontAttributeName :            [NSFont fontWithName:@"Arial" size:12],
  NSForegroundColorAttributeName : fontColor
};

//错误，冒号前应该有一个空格
NSDictionary *wrong = @{
  AKey:       @"b",
  BLongerKey: @"c",
};

//错误，每一个元素要么单独成为一行，要么全部写在一行内
NSDictionary *alsoWrong= @{ AKey : @"a",
                            BLongerKey : @"b" };

//错误，在冒号前只能有一个空格，冒号后才可以考虑按照Value对齐
NSDictionary *stillWrong = @{
  AKey       : @"b",
  BLongerKey : @"c",
};
```

#### <a name="三元运算符"></a> 三元运算符  [↩](#目录)
三元运算符 ? 应该只用在它能让代码更加清楚的地方。 一个条件语句的所有的变量应该是已经被求值了的。类似 if 语句，计算多个条件子句通常会让语句更加难以理解。或者可以把它们重构到实例变量里面。

```Objective-C
//推荐
result = a > b ? x : y;

//不推荐
result = a > b ? x = c > d ? c : d : y;
```

当三元运算符的第二个参数（if 分支）返回和条件语句中已经检查的对象一样的对象的时候，下面的表达方式更灵巧：

```Objective-C
//推荐
result = object ? : [self createObject];

//不推荐
result = object ? object : [self createObject];
```
