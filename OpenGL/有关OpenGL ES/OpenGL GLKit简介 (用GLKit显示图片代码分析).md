本文仅用作自己学习笔记

####概述
GLKit 框架的设计⽬标是为了简化基于 OpenGL / OpenGL ES 的应用开发。它的出现加快 OpenGL ES 或 OpenGL 应⽤程序开发。使⽤数学库，背景纹理加载，预先创建的着色器效果，以及标准视图和视图控制器来实现渲染循环。

* 优点
GLKit 框架提供了功能和类，可以减少创建新的基于着色器的应⽤程序所需的⼯作量，或者支持依赖早期版本的 OpenGL ES 或 OpenGL 提供的固定函数顶点或片段处理的现有应用程序。
简单明了的来讲，GLKit 就是为了让 iOS 开发者在使用OpenGL ES 或 OpenGL 的时候更简便更容易上手，所封装的库，我们直接只写核心代码就行了。

**虽然苹果弃用 OpenGL ES ，但 iOS 开发者可以继续使用。**

####用GLKit进行视图渲染, 以及部分GLKit API简介
1. EAGLContext *context;
```
//初始化视图
- (instancetype)initWithFrame:(CGRect)frame context:(EAGLContext *)context;

//配置帧缓冲区对象
drawableColorFormat 颜色缓冲区 的格式
drawableDepthFormat 深度缓冲区 的格式
drawableStencilFormat 模板缓冲区 的格式
drawableMultisample 多重采样缓冲区 的格式

// 存储绘制视图内容时使用的 OpenGL ES 上下文状态。
context

- (void)setUpConfig {
    // 1.初始化上下文&设置当前上下文
    context = [[EAGLContext alloc]initWithAPI:kEAGLRenderingAPIOpenGLES3];
    //判断context是否创建成功
    if (!context) {
        NSLog(@"Create ES context Failed");
    }
    //设置当前上下文
    [EAGLContext setCurrentContext:context];
    
    //2.获取GLKView & 设置context
    GLKView *view =(GLKView *) self.view;
    view.context = context;
        
    //3.配置视图创建的渲染缓存区.
    //GLKViewDrawableColorFormatRGBA8888 = 0，
    //默认缓存区的每个像素的最小组成部分（RGBA）使用 8 个 bit，（所以每个像素 4 个字节，4 * 8 个 bit）。
    //GLKViewDrawableColorFormatRGB565，如果你的 APP 允许更小范围的颜色，即可设置这个。会让你的 APP 消耗更小的资源（内存和处理时间）
    view.drawableColorFormat = GLKViewDrawableColorFormatRGBA8888;

    //GLKViewDrawableDepthFormatNone = 0，意味着完全没有深度缓冲区
    //GLKViewDrawableDepthFormat16，
    //GLKViewDrawableDepthFormat24，
    //如果你要使用这个属性（一般用于 3D 游戏），你应该选择GLKViewDrawableDepthFormat16 或 GLKViewDrawableDepthFormat24。这里的差别是使用 GLKViewDrawableDepthFormat16 将消耗更少的资源。

    view.drawableDepthFormat = GLKViewDrawableDepthFormat16;
    
    //4.设置背景颜色
    glClearColor(1, 0, 0, 1.0);
}
```
2. setUpVertexData
```
- (void)setUpVertexData {
    // 1.设置顶点数组(顶点坐标,纹理坐标)
    GLfloat vertexData[] = {
        0.5, -0.5, 0.0f,    1.0f, 0.0f, //右下
        0.5, 0.5, -0.0f,    1.0f, 1.0f, //右上
        -0.5, 0.5, 0.0f,    0.0f, 1.0f, //左上
        
        0.5, -0.5, 0.0f,    1.0f, 0.0f, //右下
        -0.5, 0.5, 0.0f,    0.0f, 1.0f, //左上
        -0.5, -0.5, 0.0f,   0.0f, 0.0f, //左下
    };
    //2.开辟顶点缓存区
    //(1).创建顶点缓存区标识符ID
    GLuint bufferID;
    glGenBuffers(1, &bufferID);
    //(2).绑定顶点缓存区.(明确作用)
    glBindBuffer(GL_ARRAY_BUFFER, bufferID);
    //(3).将顶点数组的数据copy到顶点缓存区中(GPU显存中)
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertexData), vertexData, GL_STATIC_DRAW);
    
    //3.打开读取通道.
    //顶点坐标数据
    glEnableVertexAttribArray(GLKVertexAttribPosition);
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * 5, (GLfloat *)NULL + 0);
    
    //纹理坐标数据
    glEnableVertexAttribArray(GLKVertexAttribTexCoord0);
    glVertexAttribPointer(GLKVertexAttribTexCoord0, 2, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * 5, (GLfloat *)NULL + 3);
}
```

在 iOS 中, 默认情况下，出于性能考虑，所有顶点着色器的属性（Attribute）变量都是关闭的。
意味着，顶点数据在着色器端（服务端）是不可用的。即使你已经使用 glBufferData 方法,将顶点数据从内存拷贝到顶点缓存区中（GPU 显存中）。
所以，必须由 glEnableVertexAttribArray 方法打开通道，指定访问属性，才能让顶点着色器能够访问到从 CPU 复制到 GPU 的数据。

注意：数据在 GPU 端是否可见，即着色器能否读取到数据，由是否启用了对应的属性决定，这就是glEnableVertexAttribArray的功能，允许顶点着色器读取 GPU（服务器端）数据。
```
//上传顶点数据到显存的方法（设置合适的方式从buffer里面读取数据）
glVertexAttribPointer (GLuint indx, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const GLvoid* ptr)

//1、indx 参数：指定要修改的顶点属性的索引值

//2、size 参数：每次读取数量（步长）。（如 position 是由3个（x,y,z）组成，而颜色是4个（r,g,b,a）,纹理则是2个）

//3、type 参数：指定数组中每个组件的数据类型。可用的符号常量有 GL_BYTE，GL_UNSIGNED_BYTE，GL_SHORT，GL_UNSIGNED_SHORT，GL_FIXED, 和 GL_FLOAT，初始值为 GL_FLOAT。

//4、normalized 参数：指定当被访问时，固定点数据值是否应该被归一化（GL_TRUE）或者直接转换为固定点值（GL_FALSE），一般设为 GL_FALSE。

//5、stride 参数：指定连续顶点属性之间的偏移量。如果为 0，那么顶点属性会被理解为：它们是紧密排列在一起的。初始值为 0

//6、ptr 参数：指定一个指针，指向数组中第一个顶点属性的第一个组件。初始值为0
```


3. GLKBaseEffect
GLKBaseEffect 是 GLKit 提供的一种简单的光照/着色系统，用于基于着色器 OpenGL 渲染。
```
//因为纹理原点是：左下角（0，0）；
//view 原点是：左上角（0，0）；
//所以在设置纹理的 options 参数时，需要传GLKTextureLoaderOriginBottomLeft 翻转一下，不然纹理就是倒着的。这是 GLKit 里的解决办法，在 OpenGL ES 里就没这么方便了。

- (void)setUpTexture {
    //1.获取纹理图片路径
    NSString *filePath = [[NSBundle mainBundle]pathForResource:@"凡几多" ofType:@"jpg"];
    
    //2.设置纹理参数
    //纹理坐标原点是左下角,但是图片显示原点应该是左上角.
    NSDictionary *options = [NSDictionary dictionaryWithObjectsAndKeys:@(1),GLKTextureLoaderOriginBottomLeft, nil];
    
    GLKTextureInfo *textureInfo = [GLKTextureLoader textureWithContentsOfFile:filePath options:options error:nil];
    
    //3.使用苹果GLKit 提供GLKBaseEffect 完成着色器工作(顶点/片元)
    cEffect = [[GLKBaseEffect alloc]init];
    cEffect.texture2d0.enabled = GL_TRUE;
    cEffect.texture2d0.name = textureInfo.name;
}
```
4. 实现代理方法 GLKViewDelegate
```
//绘制视图的内容
- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect {
    //1.
    glClear(GL_COLOR_BUFFER_BIT);
    //2.准备绘制
    [cEffect prepareToDraw];
    //3.开始绘制，用三角形，从第0个顶点开始画，一共画6个
    glDrawArrays(GL_TRIANGLES, 0, 6);
}
```