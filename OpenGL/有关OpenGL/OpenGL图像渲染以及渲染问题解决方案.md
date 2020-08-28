还是觉得做一个笔记, 已方便以后自己查阅.
[文章参考](https://www.jianshu.com/p/7b7b3250e813)
## 隐藏面消除是什么？
> 在绘制3D场景的时候，我们需要决定哪些部分是对观察者可见的，或者哪些部分是对观察者不可见的，对于不可见的部分，应该及早丢弃。例如在一个不透明的墙壁后，就不应该有渲染，这种情况叫做隐藏面消除(Hidden surface elimination).

## 解决渲染问题的方法分析
### 1. 油画算法
> 算法原理：先绘制场景中离观察者较远的物体,再绘制较近的物体.

![油画算法原理](https://upload-images.jianshu.io/upload_images/1367029-bb3feff533d510ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

绘制顺序依次是红、黄、灰，这样的话按序渲染能过解决隐藏面消除的问题.
* 效率很低，重叠部分会进行多次绘制渲染，浪费资源
还有一个问题, 如下图:
![油画算法的弊端.png](https://upload-images.jianshu.io/upload_images/1367029-447ba23e2b76fd82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.正背面剔除(Face Culling)
> 正/背面剔除原理：我们不去绘制那些根本看不到的面，以某种方式去丢弃这部分数据。
我们可以给平面定义正面和背面，OpenGL可以做到检查所有正面朝向观察者的面，并渲染它们，从而丢弃背⾯朝向的面。
OpenGL渲染的性能即可提⾼超过50%。

至于哪个是正面 哪个是背面, 我[上篇文章](https://www.jianshu.com/p/7d0391399d10)有分析, 在这里给出图解
![8033142-cad75b06dc3fc71e.png](https://upload-images.jianshu.io/upload_images/1367029-e91931da1de4fcfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 正面: 按照逆时针顶点连接顺序的三⻆形⾯
* 背⾯: 按照顺时针顶点连接顺序的三角形⾯
![8033142-f22b4230a6dc1710.png](https://upload-images.jianshu.io/upload_images/1367029-9b3bd7bae1570d9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分析:
1. 左侧三角形顶点顺序为: 1—> 2—> 3 ; 右侧三角形的顶点顺序为: 1—> 2—> 3
2. 当观察者在右侧时,则右边的三角形方向为逆时针方向,即为正面,而左侧的三⻆形为顺时针,即为背面。
3. 当观察者在左侧时,则左边的三角形方向为逆时针方向,即为正面,而右侧的三⻆形为顺时针,即为背面。
总结:
正⾯和背面是由三角形的顶点顺序和观察者方向共同决定的,随着观察者的角度方向的改变,正面背面也会跟着改变。

```
//正/背面剔除代码实现
//开启表面剔除(默认背面剔除)
void glEnable(GL_CULL_FACE);
//关闭表面剔除(默认背面剔除)
void glDisable(GL_CULL_FACE);
//用户选择剔除那个面(正面/背面)
void glCullFace(GLenum mode);
mode参数为: GL_FRONT,GL_BACK,GL_FRONT_AND_BACK ,默认GL_BACK
//用户指定顺序为正面
void glFrontFace(GLenum mode);
//mode参数为: GL_CW,GL_CCW,默认值:GL_CCW
```
### 3. 上面的正背面剔除还不够, 还要开启深度测试
>1､什么是深度？
深度其实就是该像素点在3D世界中距离摄像机的距离,Z值。
>2､什么是深度缓冲区?
深度缓存区,就是一块内存区域,专门存储着每个像素点(绘制在屏幕上的)深度值.深度值(Z值)越大, 则离摄像机就越远。
>3､为什么需要深度缓冲区?
在不使用深度测试的时候,如果我们先绘制一个距离比较近的物体,再绘制距离较远的物体,则距离远的位图因为后绘制,会把距离近的物体覆盖掉. 有了深度缓冲区后,绘制物体的顺序就不那么􏰁重要了. 实际上,只要存在深度缓冲区,OpenGL都会把像素的深度值写入到缓冲区中. 除⾮调用glDepthMask(GL_FALSE)来禁⽌写⼊。
>4､深度测试
深度缓冲区(DepthBuffer)和颜色缓存区(ColorBuffer)是对应的，颜色缓存区存储像素的颜色信息，而深度缓冲区存储像素的深度信息。在决定是否绘制⼀个物体表面时, ⾸先要将表面对应的像素的深度值与当前深度缓冲区中的值进⾏行比较. 如果大于深度缓冲区中的值,则丢弃这部分.否则利用这个像素对应的深度值和颜色值分别更新深度缓冲区和颜色缓存区. 这个过程称为”深度测试”。

* 深度缓冲区,一般由窗口管理系统GLFW创建.深度值一般由16位、24位、32位值表示. 通常是24位.位数越高,深度精确度更好。
```
开启深度测试
glEnable(GL_DEPTH_TEST);
在绘制场景前,清除颜色缓存区,深度缓冲区
glClearColor(0.0f,0.0f,0.0f,1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
打开/阻断深度缓存区写入
void glDepthMask(GLBool value);
value : GL_TURE 开启深度缓冲区写入; GL_FALSE 关闭深度缓冲区写入
```

* 指定深度测试判断模式void glDepthFunc(GLEnum mode);
![8033142-e62fd21fda064938.png](https://upload-images.jianshu.io/upload_images/1367029-75614ce6bec2c555.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## ZFighting闪烁问题
>为什么会出现ZFighting闪烁问题
因为开启深度测试后，OpenGL就不会去绘制模型被遮挡的部分，这样实现现实更加真实，但是由于深度缓存区精度的限制，对于深度相差无几的情况下，OpenGL就可能出现不能正确判断两者深度值，会导致深度测试的结果不可预测，现实出来的现象会交错闪烁。

![11278703-ba2006df6f7eae19.png](https://upload-images.jianshu.io/upload_images/1367029-2c639318f808703c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
解决方法
1. 第一步：启用Polygon Offset方式解决
让深度值之间产生间隔，可以理解为在执行深度测试前，将立方体的深度值做一些细微的增加，于是就能将重叠的2个图形深度值之间有所区分。
```
// 启用Polygon Offset方式
glEnable(GL_POLYGON_OFFSET_FILL);

参数列表：
GL_POLYGON_OFFSET_POINT 对应光栅化模式：GL_POINT
GL_POLYGON_OFFSET_LINE 对应光栅化模式：GL_LINE
GL_POLYGON_OFFSET_FILL 对应光栅化模式：GL_FILL
```
2. 第二步指定偏移量
* 通过glPolygon Offset 来指定. glPolygon Offset需要2个参数: factor , units.
* 每个Fragment 的深度值都会增加如下所示的偏移量:
Offset = ( m * factor ) + ( r * units); 
m : 多边形的深度的斜率的最大值，理解一个多边形越是与近裁剪⾯平行，m就越接近于0.
r : 能产生于窗口坐标系的深度值中可分辨的差异最小值.r是由具体是由具体OpenGL平台指定的一个常量.
* 一个⼤于0的Offset会把模型推到离你(摄像机)更远的位置，相应的⼀个小于0的Offset 会把模型拉近
* 一般⽽言，只需要将-1.0 和 -1 这样简单赋值给glPolygon Offset 基本可以满⾜足需求

3. 第三步: 关闭 ` Polygon Offset `
```
glDisable(GL_POLYGON_OFFSET_FILL);
```