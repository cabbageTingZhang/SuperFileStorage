## OpenGL渲染管线简化流程图
以下图片转自[OpenGL基础渲染](https://www.jianshu.com/p/c0e71b4c1bf4)
![管线渲染流程图](https://upload-images.jianshu.io/upload_images/1367029-2838a8dfb837c832.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1. 客户端-服务器
这里的对于OpenGL而言, 客户端是存储在CPU中的代码, 驱动程序将渲染命令与数据组合起来发给服务器执行.
而Server调用的就是GPU芯片的意思.
服务器和客服端在功能上是异步的, 客服端不断的将数据和命令组合在一起送入缓冲区, 缓冲区再发送的服务器执行.

### 2. 着色器
上图中最大的框架代表是 **顶点着色器** 和 **片元着色器**.  (GLSL编写的)
**顶点着色器**: 顶点着色器处理从客服端中输入的数据, 用数学运算来计算光照效应, 位移, 颜色值等. 有几个顶点, 顶点着色器就要执行几次.
**图元组合(Primitive Assembly)**: 上面的框图说明3个顶点已经组合在一起了.
**片元着色器**: 片元着色器用来计算片元的最终颜色 (尽管在下一个阶段(逐片元的操作)时可能还会改变颜色一次)和它的深度值. 在这里我们会使用纹理映射的方式, 对顶点处理阶段所计算的颜色值进行补充. 如果我们觉得不应该继续绘制某个片元, 在片元着色器中还可以终止这个片元的处理, 这一步叫做片元的丢弃(discard).
顶点着色器和片元着色器之间的区别: 顶点着色(包括细分和几何着色)决定了一个图元应该位于屏幕的什么位置, 而片元着色器使用这些信息来决定某个片元的颜色应该是什么.

### 3. 三种向OpenGL着色器传递渲染数据的方法

1. 属性：就是对⼀个顶点都要作出改变的数据元素。实际上，顶点位置本身就是一个属性.。属性可以是浮点类型，整型，布尔类型等。
2. Uniform值：通过设置 Uniform 变量就紧接着发送一个图元批次处理命令。Uniform 变量实际上可以无限次的使⽤。 设置一个应用于整个表⾯面的单个颜色值，还可也是一个时间值。
```
//代码段 并不连续 仅做参考
// Use a stock shader, and pass in the parameters needed
GLint UseStockShader(GLT_STOCK_SHADER nShaderID, ...);
// 定义一个颜色
GLfloat vBlack[] = { 0.0f, 0.0f, 0.0f, 1.0f };
//声明一个全局的存储着色器变量
GLShaderManager     shaderManager;
//以下就是用法
shaderManager.UseStockShader(GLT_SHADER_FLAT, transformPipeline.GetModelViewProjectionMatrix(), vBlack);
//使用 单位着色器
//参数1：简单的使用默认笛卡尔坐标系（-1，1），所有片段都应用一种颜色。GLT_SHADER_IDENTITY
//参数2：着色器颜色
shaderManager.UseStockShader(GLT_SHADER_IDENTITY, vGreen);
//函数声明
GLShaderManger::UseStockShader(GLT_SHADER_IDENTITY,GLfloat mvp[16],GLfloat vColor[4])
```
3. 纹理：对纹理进行采样和筛选。纹理数据的作用不仅仅是表现图形。很多图形文件格式都是以无符号字节形式对颜色分量进行存储的，但我们仍然可以设置浮点纹理。这就是说，任何大型浮点数据块（例如消耗资源很大的函数的大型查询表）都可以通过这种方式传递给着色器。

## 着色器的使用以及图元

标识符 | 描述
 ----  | ----
GLT_ATTRIBUTE_VERTEX |	3分量（x, y, z）顶点位置
GLT_ATTRIBUTE_COLOR  |	4分量（r, g, b, a）颜色值
GLT_ATTRIBUTE_NORMAL | 3分量（x, y, z）表面法线
GLT_ATTRIBUTE_TEXTURE0 | 第一对 2 分量（s ,t）纹理坐标
GLT_ATTRIBUTE_TEXTURE1 | 第二对 2 分量（s ,t）纹理坐标

```
//单位（Identity）着色器 GLT_SHADER_IDENTITY
参数1：单位着色器
参数2：颜色值
GLShaderManager::UseStockShader(GLT_SHADER_IDENTITY, GLfoat vColor[4]);

//平面（Flat）着色器 GLT_SHADER_FLAT
参数1：平面着色器
参数2：允许变化的4*4矩阵
参数3：颜色值
GLShaderManager::UseStockShader(GLT_SHADER_FLAT, GLfoat mvp[16], GLfloat vColor[4]);

//上色（Shaded）着色器 GLT_SHADER_SHADED
GLShaderManager::UseStockShader(GLT_SHADER_SHADED, GLfoat mvp[16]);

// 默认光源着色器 GLT_SHADER_DEFAULT_LIGHT
参数1：默认光源着色器
参数2：模型视图矩阵
参数3：投影矩阵
参数4：颜色值
GLShaderManager::UseStockShader(GLT_SHADER_DEFAULT_LIGHT, GLfoat mvMatrix[16], GLfloat pMatrix[16], GLfloat vColor[4]);

//点光源着色器 GLT_SHADER_POINT_LIGHT_DIFF
参数1：点光源着色器
参数2：模型视图矩阵
参数3：投影矩阵
参数4：视点坐标系中的光源位置
参数5：颜色值
GLShaderManager::UseStockShader(GLT_SHADER_POINT_LIGHT_DIFF, GLfoat mvMatrix[16], GLfloat pMatrix[16], GLfloat vLightPos[3], GLfloat vColor[4]);

//纹理替换矩阵 GLT_SHADER_TEXTURE_REPLACE
GLShaderManager::UseStockShader(GLT_SHADER_TEXTURE_REPLACE, GLfoat mvMatrix[16], GLint nTextureUnit);

//纹理调整着色器 GLT_SHADER_TEXTURE_MODULATE
GLShaderManager::UseStockShader(GLT_SHADER_TEXTURE_MODULATE, GLfoat mvMatrix[16], GLfloat vColor, GLint nTextureUnit);

//纹理光源着色器 GLT_SHADER_TEXTURE_POINT_LIGHT_DIFF
参数1：纹理光源着色器
参数2：模型视图矩阵
参数3：投影矩阵
参数4：视点坐标系中的光源位置
参数5：几何图形的基本色
参数6：将要使用的纹理单元
GLShaderManager::UseStockShader(GLT_SHADER_TEXTURE_POINT_LIGHT_DIFF, GLfloat mvMatrix, GLfoat mvMatrix[16], GLfloat vLightPos[3], GLfloat vBaseColor[4], GLint nTextureUnit);
```

### OpenGL图元的模式标识

图元类型 | OpenGL 枚举量
---- | ----
点 | GL_POINTS
线 | GL_LINES
条带线 | GL_LINE_STRIP
循环线 | GL_LINE_LOOP
独立三角形 | GL_TRIANGLES
三角形条带 | GL_TRIANGLE_STRIP
三角形扇面 | GL_TRIANGLE_FAN

#### 环绕
将顺时针方向绘制的三角形用逆时针的方式绘制。
如下图，在绘制第一个三角形时，线条是按照从V0-V1，再到V2。最后再回到V0的一个闭合三角形。 这个是沿着顶点顺时针方向。
这种顺序与方向结合来指定顶点的方式称为 环绕。
下图的2个三角形的环绕方向完全相反。
![三角形环绕](https://upload-images.jianshu.io/upload_images/1367029-830c135b0d14740f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 正面与背面:
在默认的情况下，OpenGL认为具有逆时针方向环绕的多边形是 正面的。而右侧的顺时针方向三角形是三角形的 背面。
> **为什么区分正背面很重要？**
因为我们常常希望为一个多边形的正面和背面分别设置不同的物理特征。我们可以完全隐藏一个多边形的背面，或者给它设置一种不同的颜色和反射属性。纹理图像在背面三角形中也是相反的。在一个场景中，使所有的多边形保持环绕方向的一致，并使用正面多边形来绘制所有实心物体的表面是非常重要的。

```
//如果想改变OpenGL这个默认行为，可以调用下面的函数：
glFrontFace(GL_CW);
参数：GL_CW | GL_CCW
GL_CCW：表示传入的mode会选择逆时针为前向
GL_CW：表示顺时针为前向。
默认：GL_CCW。逆向时针为前向。
```

#### 有关三角形带的说明
对于很多表面和形状来说，我们可能需要绘制几个相连的三角形。我们可以使用GL_TRIANGLE_STRIP图元绘制一串相连的三角形。从而节省大量的时间。
![1802326-73bce5a2ffe3116a.png](https://upload-images.jianshu.io/upload_images/1367029-399beb6069519e66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 有关三角形带的优点
> 1. 用前3个顶点指定第1个三角形之后，对于接下来的每一个三角形，只需要再指定1个顶点。需要绘制大量的三角形时，采用这种方法可以节省大量的程序代码和数据存储空间。
> 2. 提供运算性能和节省带宽。更少的顶点意味着数据从内存传输到图形卡的速度更快，并且顶点着色器需要处理的次数也更少了。