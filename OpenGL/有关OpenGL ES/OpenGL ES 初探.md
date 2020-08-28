#### 简介
[文章图片出处(仅用于笔记记录)](https://www.jianshu.com/p/f58fff6d0ba0)
> OpenGL ES 开放式图形库(OpenGL的)用于可视化的二维和三维数据。它是⼀个多功能开放标准图形库，支持2D和3D数字内 容创建，机械和建筑设计，虚拟原型设计，飞行模拟，视频游戏等应⽤用程序。您可以使⽤用OpenGL配置3D图形管道并向其提交数据。顶点被变换和点亮，组合成图元，并光栅化以创建2D图像。OpenGL旨在将函数调⽤用转换为可以发送到底层图形硬件的图形命令。由于此底层硬件专⽤于处理理图形命令，因此OpenGL绘图通常⾮非常快。

> OpenGL for Embedded Systems(OpenGL ES)是OpenGL的简化版本，它消除了了冗余功能，提供了了⼀一个既易于学习又更易于在移动图形硬件 中实现的库。

#### 关键步骤介绍
##### 顶点着色器
1. 着⾊器程序—描述顶点上执⾏操作的顶点着⾊器程序源代码/可执⾏⽂件 
2. 顶点着⾊器输入(属性) — 用顶点数组提供每个顶点的数据 
3. 统⼀变量(uniform)—顶点/⽚元着色器使用的不变数据
4. 采样器—代表顶点着色器使用纹理的特殊统一变量类型.

![顶点着色器](https://upload-images.jianshu.io/upload_images/1367029-81434fb1aa2f133e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 顶点着色器所做的主要业务有
1. 矩阵变换位置 
2. 计算光照公式⽣生成逐顶点颜⾊色
3. ⽣生成/变换纹理理坐标

总结: 它可以⽤于执行自定义计算,实施新的变换,照明或者传统的固定功能所不允许的基于顶点的效果.

* 顶点着色器代码参考
```
//attribute 和 uniform 对应 通道修饰符 ，
//分别是 attribute属性通道 和 uniform 通道。
//vec4、vec2 和 mat4 对应 类型，vec4 代表 4 维向量，vec2 代表 2 维向量，
//mat4 代表 4 行 4 列的矩阵类型
attribute vec4 position;
attribute vec2 textCoordinate;
uniform mat4 rotateMatrix;
varying lowp vec2 varyTextCoord;
void main()
{
// 把textCoordinate交给varyTextCoord，就可以把纹理坐标传递到片元着色器里面去。
    varyTextCoord = textCoordinate;

    vec4 vPos = position;
    vPos = vPos * rotateMatrix;// 让每一个顶点都和旋转矩阵相乘
    
    gl_Position = vPos;// gl_Position是一个内建变量，是vec4类型的，必须给它赋值。
}
```

##### 图元装配
> 顶点着色器之后，OpenGL ES 3.0 图形管线的下一阶段是 图元装配。
图元（Primitive）：是三角形、直线或者点精灵等几何对象。
图元装配：将顶点数据计算组合成一个个图元，在这个阶段会执行裁剪、透视分割和 Viewport 变换操作。

在这之后将进⼊光栅化阶段。

##### 光栅化
在此阶段绘制对应的图元（点精灵、直线或者三角形）。
光栅化是将图元转化为一组二维片段的过程，然后这些片段由片元着色器处理。这些二维片段代表着可在屏幕上绘制的像素。
![光栅化阶段](https://upload-images.jianshu.io/upload_images/1367029-b51525b764366570.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 片元着色器阶段
1. 着色器程序 —— 描述⽚段上执⾏操作的片元着⾊器程序源代码/可执行⽂件。
2. 输入变量—— 光栅化单元用插值为每个片段生成的顶点着⾊器输出。
3. 统一变量（uniform）—— 顶点（或者片段）着色器使用的不变数据。
4. 采样器 —— 代表⽚元着色器使⽤纹理的特殊统一变量类型。
![片元着色器](https://upload-images.jianshu.io/upload_images/1367029-faff71525d57527c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 片元着色器的主要业务
1. 计算颜⾊
2. 获取纹理值
3. 往像素点中填充颜⾊值(纹理值/颜⾊值);

总结: 它可以⽤用于图片/视频/图形中每个像素的颜色填充(⽐如给视频添加滤镜,实际 上就是将视频中每个图片的像素点颜色填充进行修改.)

片元着色器代码案例
```
//顶点着色器 里的 varying lowp vec2 varyTextCoord和片元着色器里的 varying lowp vec2 varyTextCoord必须定义的一模一样才行，包括通道、精度、类型和变量名都要一致，这样才能从顶点着色器传进来。
//uniform sampler2D colorMap 是由 uniform 通道传进来的纹理采样器，通过它可以拿到对应的纹理。
varying lowp vec2 varyTextCoord;
uniform sampler2D colorMap;
void main()
{
    // texture2D(纹理采样器, 纹理坐标); 为了取得纹素（纹理对应坐标上的颜色值），比如取到了一个rgba四维变量
    gl_FragColor = texture2D(colorMap, varyTextCoord);
}
```

##### EGL (Embedded Graphics Library )
* OpenGL ES 命令需要渲染上下文和绘制表面才能完成图形图像的绘制. 
* 渲染上下文: 存储相关OpenGL ES 状态(状态机). 
* 绘制表面: 是用于绘制图元的表面,它指定渲染所需要的缓存区类型,例如颜⾊缓存 区,深度缓冲区和模板缓存区.
* OpenGL ES API 并没有提供如何创建渲染上下文或者上下⽂如何连接到原⽣窗口系统. EGL 是Khronos 渲染API(如OpenGL ES) 和原生窗口系统之间的接口. 唯⼀支持 OpenGL ES 却不支持EGL 的平台是iOS. Apple 提供⾃⼰的EGL API的iOS实现,称为EAGL.
* 因为每个窗口系统都有不同的定义,所以EGL提供基本的不透明类型—EGLDisplay, 这 个类型封装了所有系统相关性,用于和原生窗⼝系统接⼝.


