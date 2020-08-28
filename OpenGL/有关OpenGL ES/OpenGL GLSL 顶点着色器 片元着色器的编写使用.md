> 前言:  有关使用GLSL自己去编写顶点着色器和片元着色器, 因为没有编译器支持提示, 所以很容易写错, 本篇文章介绍自定义编写的代码, 以及相应的使用和语法知识.

在自定义的着色器里面尽量不加中文注释, 可能会导致编译失败, 以下根据两端代码来进行分析: **注意以下代码在代码片段中有中文注释**
* 顶点着色器
```
//顶点坐标
attribute vec4 position;
//纹理坐标
attribute vec2 textCoordinate;
//纹理坐标
varying lowp vec2 varyTextCoord;

void main()
{
    //通过varying 修饰varyTextCoord, 将纹理坐标传递到片源着色器
    varyTextCoord = textCoordinate;
    //gl_Position作为顶点着色器的内置函数, 要对他进行赋值, 顶点着色器才会起作用.
    gl_Position = position;
}
```

varyTextCoord 用varying 修饰, 这种写法作为顶点跟片源着色器之间传值使用的, 片源着色器要想接收到该值, 在片元着色器中声明的属性 必须跟在顶点中声明的完全一样.

* 片元着色器

```
//纹理坐标
varying lowp vec2 varyTextCoord;
//纹理采样器(获取对应的纹理ID)
uniform sampler2D colorMap;

void main()
{
    //texture2D(纹理采样器, 纹理坐标), 获取对应坐标纹素
    //gl_FragColor GLSL 内建变量(赋值像素点颜色值)
    //纹理颜色添加对应像素点上
    //gl_FragColor 内建变量. GLSL语言已经提前定义好的变量, 有相应特殊函数.
    //内建函数. GLSL语言提前封装好的相关函数.
    //读取纹素, vec4 texture2D(纹理colorMap, 纹理坐标varyTextCoord); rgba
    lowp vec4 temp = texture2D(colorMap, varyTextCoord);
    gl_FragColor = temp;
}
```

最终我们在GLSL代码中需要的是一个**GLuint myPrograme**的对象.
有关着色器文件的命名, 顶点着色器尽量用.vsh, 片元着色器尽量用.fsh.
vsh/fsh -> program -> GPU

以下增加vsh/fsh -> program的代码
``` Objective-C
//加载shader
-(GLuint)loadShaders:(NSString *)vert Withfrag:(NSString *)frag
{
    //1.定义2个零时着色器对象
    GLuint verShader, fragShader;
    //创建program
    GLint program = glCreateProgram();
    
    //2.编译顶点着色程序、片元着色器程序
    //参数1：编译完存储的底层地址
    //参数2：编译的类型，GL_VERTEX_SHADER（顶点）、GL_FRAGMENT_SHADER(片元)
    //参数3：文件路径
    [self compileShader:&verShader type:GL_VERTEX_SHADER file:vert];
    [self compileShader:&fragShader type:GL_FRAGMENT_SHADER file:frag];
    
    //3.创建最终的程序
    glAttachShader(program, verShader);
    glAttachShader(program, fragShader);
    
    //4.释放不需要的shader
    glDeleteShader(verShader);
    glDeleteShader(fragShader);
    
    return program;
}

//编译shader
- (void)compileShader:(GLuint *)shader type:(GLenum)type file:(NSString *)file{
    
    //1.读取文件路径字符串
    NSString* content = [NSString stringWithContentsOfFile:file encoding:NSUTF8StringEncoding error:nil];
    const GLchar* source = (GLchar *)[content UTF8String];
    
    //2.创建一个shader（根据type类型）
    *shader = glCreateShader(type);
    
    //3.将着色器源码附加到着色器对象上。
    //参数1：shader,要编译的着色器对象 *shader
    //参数2：numOfStrings,传递的源码字符串数量 1个
    //参数3：strings,着色器程序的源码（真正的着色器程序源码）
    //参数4：lenOfStrings,长度，具有每个字符串长度的数组，或NULL，这意味着字符串是NULL终止的
    glShaderSource(*shader, 1, &source,NULL);
    
    //4.把着色器源代码编译成目标代码
    glCompileShader(*shader);
}

```

以下代码为program的使用 以及编译
```
//3.加载shader
    self.myPrograme = [self loadShaders:vertFile Withfrag:fragFile];
    
    //4.链接
    glLinkProgram(self.myPrograme);
    GLint linkStatus;
    //获取链接状态
    glGetProgramiv(self.myPrograme, GL_LINK_STATUS, &linkStatus);
    if (linkStatus == GL_FALSE) {
        GLchar message[512];
        glGetProgramInfoLog(self.myPrograme, sizeof(message), 0, &message[0]);
        NSString *messageString = [NSString stringWithUTF8String:message];
        NSLog(@"Program Link Error:%@",messageString);
        return;
    }
    
    NSLog(@"Program Link Success!");
    //5.使用program
    glUseProgram(self.myPrograme);
```

最后请看怎么访问着色器中的属性, 因为着色器最后合并生成了programe, 所以具体查看一下代码
```
//8.将顶点数据通过myPrograme中的传递到顶点着色程序的position
    //1.glGetAttribLocation,用来获取vertex attribute的入口的.
    //2.告诉OpenGL ES,通过glEnableVertexAttribArray，
    //3.最后数据是通过glVertexAttribPointer传递过去的。
    
    //(1)注意：第二参数字符串必须和shaderv.vsh中的输入变量：position保持一致
    GLuint position = glGetAttribLocation(self.myPrograme, "position");
    
    //(2).设置合适的格式从buffer里面读取数据
    glEnableVertexAttribArray(position);
    
    //(3).设置读取方式
    //参数1：index,顶点数据的索引
    //参数2：size,每个顶点属性的组件数量，1，2，3，或者4.默认初始值是4.
    //参数3：type,数据中的每个组件的类型，常用的有GL_FLOAT,GL_BYTE,GL_SHORT。默认初始值为GL_FLOAT
    //参数4：normalized,固定点数据值是否应该归一化，或者直接转换为固定值。（GL_FALSE）
    //参数5：stride,连续顶点属性之间的偏移量，默认为0；
    //参数6：指定一个指针，指向数组中的第一个顶点属性的第一个组件。默认为0
    glVertexAttribPointer(position, 3, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * 5, NULL);
    
    
    //9.----处理纹理数据-------
    //(1).glGetAttribLocation,用来获取vertex attribute的入口的.
    //注意：第二参数字符串必须和shaderv.vsh中的输入变量：textCoordinate保持一致
    GLuint textCoor = glGetAttribLocation(self.myPrograme, "textCoordinate");
    
    //(2).设置合适的格式从buffer里面读取数据
    glEnableVertexAttribArray(textCoor);
    
    //(3).设置读取方式
    //参数1：index,顶点数据的索引
    //参数2：size,每个顶点属性的组件数量，1，2，3，或者4.默认初始值是4.
    //参数3：type,数据中的每个组件的类型，常用的有GL_FLOAT,GL_BYTE,GL_SHORT。默认初始值为GL_FLOAT
    //参数4：normalized,固定点数据值是否应该归一化，或者直接转换为固定值。（GL_FALSE）
    //参数5：stride,连续顶点属性之间的偏移量，默认为0；
    //参数6：指定一个指针，指向数组中的第一个顶点属性的第一个组件。默认为0
    glVertexAttribPointer(textCoor, 2, GL_FLOAT, GL_FALSE, sizeof(GLfloat)*5, (float *)NULL + 3);
```


