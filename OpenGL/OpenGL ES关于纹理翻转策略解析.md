在用OpenGL ES绘制图片的时候, 我们发现所绘制的图片颠倒了, 以下我们来使用几种解析策略来解决这个问题, 通过探索找到最适合自己的方法.

##### 1. 给顶点着色器添加一个矩阵, 通过矩阵相乘来达到翻转图片的效果, 顶点着色器代码如下

```
attribute vec4 position;
attribute vec2 textCoordinate;
//rotateMatrix该属性为相应的翻转矩阵
uniform mat4 rotateMatrix;

varying lowp vec2 varyTextCoord;

void main()
{
    varyTextCoord = textCoordinate;

    vec4 vPos = position;
    vPos = vPos * rotateMatrix;
    
    gl_Position = vPos;
}
```

具体实现方法如下: 

```Objective-C
-(void)rotateTextureImage
{
    //注意，想要获取shader里面的变量，这里记得要在glLinkProgram后面，后面，后面！
    //1. rotate等于shaderv.vsh中的uniform属性，rotateMatrix
    GLuint rotate = glGetUniformLocation(self.myPrograme, "rotateMatrix");
    
    //2.获取渲旋转的弧度
    float radians = 180 * 3.14159f / 180.0f;
   
    //3.求得弧度对于的sin\cos值
    float s = sin(radians);
    float c = cos(radians);
    
    //4.因为在3D课程中用的是横向量，在OpenGL ES用的是列向量
    // 参考Z轴旋转矩阵
    GLfloat zRotation[16] = {
        c,-s,0,0,
        s,c,0,0,
        0,0,1,0,
        0,0,0,1
    };
    
    //5.设置旋转矩阵
    /*
     glUniformMatrix4fv (GLint location, GLsizei count, GLboolean transpose, const GLfloat* value)
     location : 对于shader 中的ID
     count : 个数
     transpose : 转置
     value : 指针
     */
    glUniformMatrix4fv(rotate, 1, GL_FALSE, zRotation);
}
```

##### 2. 在图片解压缩的过程中, 翻转画布
```
- (GLuint)setupTexture:(NSString *)fileName {
    CGImageRef spriteImage = [UIImage imageNamed:fileName].CGImage;
    if (!spriteImage) {
        NSLog(@"Failed to load image %@", fileName);
        exit(1);
    }
    
    size_t width = CGImageGetWidth(spriteImage);
    size_t height = CGImageGetHeight(spriteImage);
    
    GLubyte * spriteData = (GLubyte *) calloc(width * height * 4, sizeof(GLubyte));
  
    CGContextRef spriteContext = CGBitmapContextCreate(spriteData, width, height, 8, width*4,CGImageGetColorSpace(spriteImage), kCGImageAlphaPremultipliedLast);
    
    CGRect rect = CGRectMake(0, 0, width, height);
    CGContextDrawImage(spriteContext, rect, spriteImage);
   
//    CGContextTranslateCTM(spriteContext, rect.origin.x, rect.origin.y);
//    CGContextTranslateCTM(spriteContext, 0, rect.size.height);
//    CGContextScaleCTM(spriteContext, 1.0, -1.0);
//    CGContextTranslateCTM(spriteContext, -rect.origin.x, -rect.origin.y);
//    CGContextDrawImage(spriteContext, rect, spriteImage);
    
    //以下3行也可以!
    CGContextTranslateCTM(spriteContext, 0, rect.size.height);
    CGContextScaleCTM(spriteContext, 1.0, -1.0);
    CGContextDrawImage(spriteContext, rect, spriteImage);
    
    CGContextRelease(spriteContext);
    
    glBindTexture(GL_TEXTURE_2D, 0);
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR );
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR );
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    
    float fw = width, fh = height;
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, fw, fh, 0, GL_RGBA, GL_UNSIGNED_BYTE, spriteData);
    free(spriteData);   
    return 0;
}
``` 

##### 3. 修改片元着色器的代码
就是将纹理的y坐标倒置 (注意以下注释的代码行为原逻辑)

```
varying lowp vec2 varyTextCoord;
uniform sampler2D colorMap;

void main()
{
    //gl_FragColor = texture2D(colorMap, varyTextCoord);
    gl_FragColor = texture2D(colorMap, vec2(varyTextCoord.x,1.0-varyTextCoord.y));
}
```

##### 4. 修改顶点着色器
其实意思跟方法三差不多, 只不过在传给片元着色器以前就先将纹理坐标的y值进行翻转.(注意以下注释的代码行为原逻辑)

```
attribute vec4 position;
attribute vec2 textCoordinate;
varying lowp vec2 varyTextCoord;

void main()
{
    //varyTextCoord = textCoordinate;
    varyTextCoord = vec2(textCoordinate.x,1.0-textCoordinate.y);
    gl_Position = position;
}

```

##### 5. 修改顶点数组, 在给顶点数组时, 将纹理坐标翻转设置

```Objective-C
    //设置顶点、纹理坐标
    //前3个是顶点坐标，后2个是纹理坐标
//    GLfloat attrArr[] =
//    {
//        0.5f, -0.5f, -1.0f,     1.0f, 0.0f,
//        -0.5f, 0.5f, -1.0f,     0.0f, 1.0f,
//        -0.5f, -0.5f, -1.0f,    0.0f, 0.0f,
//
//        0.5f, 0.5f, -1.0f,      1.0f, 1.0f,
//        -0.5f, 0.5f, -1.0f,     0.0f, 1.0f,
//        0.5f, -0.5f, -1.0f,     1.0f, 0.0f,
//    };
    
    GLfloat attrArr[] =
    {
        0.5f, -0.5f, -1.0f,     1.0f, 1.0f,
        -0.5f, 0.5f, -1.0f,     0.0f, 0.0f,
        -0.5f, -0.5f, -1.0f,    0.0f, 1.0f,
        
        0.5f, 0.5f, -1.0f,      1.0f, 0.0f,
        -0.5f, 0.5f, -1.0f,     0.0f, 0.0f,
        0.5f, -0.5f, -1.0f,     1.0f, 1.0f,
    };
```

五种方法都可以达到想要的效果, 请根据自己的需要, 自行选择.