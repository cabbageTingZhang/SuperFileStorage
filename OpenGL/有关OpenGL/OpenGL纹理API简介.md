以下为经常使用的纹理API笔记, 仅做记录学习
##### 1. 读取文件
```
//参数1:x,矩形左下⻆角的窗⼝口坐标
//参数2:y,矩形左下⻆角的窗⼝口坐标
//参数3:width,矩形的宽，以像素为单位 
//参数4:height,矩形的⾼高，以像素为单位
//参数5:format,OpenGL 的像素格式，参考 表6-1 
//参数6:type,解释参数pixels指向数据，告诉OpenGL 使⽤用缓存区中的什么数据类型来存储颜⾊色分量量，像素数据的数据类型，参考 下图 
//参数7:pixels,指向图形数据的指针
void glReadPixels(GLint x,GLint y,GLSizei width,GLSizei height, GLenum format, GLenum type,const void * pixels);
```

![类型参考](https://upload-images.jianshu.io/upload_images/1367029-d66f6ec40206a4b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 2. 载入纹理
```
// target:`GL_TEXTURE_1D`、`GL_TEXTURE_2D`、`GL_TEXTURE_3D`。
// Level:指定所加载的mip贴图层次。⼀一般我们都把这个参数设置为0。
// internalformat:每个纹理理单元中存储多少颜⾊色成分。
// width、height、depth参数:指加载纹理理的宽度、⾼高度、深度。==注意!==这些值必须是 2的整数次⽅方。(这是因为OpenGL 旧版本上的遗留留下的⼀一个要求。当然现在已经可以⽀支持不不是 2的整数次⽅方。但是开发者们还是习惯使⽤用以2的整数次⽅方去设置这些参数。)
// border参数:允许为纹理理贴图指定⼀一个边界宽度。
// format、type、data参数:与我们在讲glDrawPixels 函数对于的参数相同
void glTexImage2D(GLenum target,GLint level,GLint internalformat,GLsizei width,GLsizei height,GLint border,GLenum format,GLenum type,void * data);
```
至于glTexImage1D,glTexImage3D因为使用的不多,请参考以上函数

##### 3. 更新纹理
```
void glTexSubImage1D(GLenum target,GLint level,GLint xOffset,GLsizei width,GLenum  format,GLenum type,const GLvoid *data);
void glTexSubImage2D(GLenum target,GLint level,GLint xOffset,GLint yOffset,GLsizei width,GLsizei height,GLenum format,GLenum type,const GLvoid *data);
void glTexSubImage3D(GLenum target,GLint level,GLint xOffset,GLint yOffset,GLint  zOffset,GLsizei width,GLsizei height,GLsizei depth,Glenum type,const GLvoid * data);
```
##### 4. 插入替换纹理
```
void glCopyTexSubImage1D(GLenum target,GLint level,GLint xoffset,GLint x,GLint y,GLsize width);
void glCopyTexSubImage2D(GLenum target,GLint level,GLint xoffset,GLint yOffset,GLint x, GLint y,GLsizei width,GLsizei height);
void glCopyTexSubImage3D(GLenum target,GLint level,GLint xoffset,GLint yOffset,GLint zOffset,GLint x,GLint y,GLsizei width,GLsizei height);
```
##### 5. 使用颜色缓存区加载数据, 形成新的纹理
```
//x,y 在颜⾊色缓存区中指定了了开始读取纹理理数据的位置; 缓存区⾥里里的数据是源缓存区通过glReadBuffer设置的。
void glCopyTexImage1D(GLenum target,GLint level,GLenum internalformt,GLint x,GLint y,GLsizei width,GLint border);
//注意:不不存在glCopyTextImage3D ，因为我们⽆无法从2D 颜⾊色缓存区中获取体积数据。
void glCopyTexImage2D(GLenum target,GLint level,GLenum  internalformt,GLint x,GLint y,GLsizei width,GLsizei height,GLint border);
```

#### 纹理对象
```
//使⽤用函数分配纹理理对象
//指定纹理理对象的数量量 和 指针(指针指向⼀一个⽆无符号整形数组，由纹理理对象标识符填充)。 
void glGenTextures(GLsizei n,GLuint * textTures);

//绑定纹理理状态 //参数target:GL_TEXTURE_1D、GL_TEXTURE_2D GL_TEXTURE_3D
//参数texture:需要绑定的纹理理对象
void glBindTexture(GLenum target,GLunit texture);

//删除绑定纹理理对象
//纹理理对象 以及 纹理理对象指针(指针指向⼀一个⽆无符号整形数组，由纹理理对象标识符填充)。
void glDeleteTextures(GLsizei n,GLuint *textures);

//测试纹理理对象是否有效
//如果texture是⼀一个已经分配空间的纹理理对象，那么这个函数会返回GL_TRUE,否则会返回GL_FALSE。 
GLboolean glIsTexture(GLuint texture);
```

#### 设置纹理参数
```
// 参数1:target,指定这些参数将要应⽤用在那个纹理理模式上，⽐如GL_TEXTURE_1D、GL_TEXTURE_2D、GL_TEXTURE_3D。
// 参数2:pname,指定需要设置那个纹理理参数
// 参数3:param,设定特定的纹理理参数的值
glTexParameterf(GLenum target,GLenum pname,GLFloat param);
glTexParameteri(GLenum target,GLenum pname,GLint param);
glTexParameterfv(GLenum target,GLenum pname,GLFloat *param);
glTexParameteriv(GLenum target,GLenum pname,GLint *param);
```

* 有关过滤方式
主要有```邻近过滤(GL_NEAREST)``` 和``` 线性过滤(GL_LINEAR)```
![左侧为邻近过滤(GL_NEAREST) 右侧为线性过滤(GL_LINEAR)](https://upload-images.jianshu.io/upload_images/1367029-a5cde1e5bbee28d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![两种过滤方式的比较](https://upload-images.jianshu.io/upload_images/1367029-f3bb0d11ff5cbba1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
//纹理理缩⼩小时,使⽤用邻近过滤
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_NEAREST) 
//纹理理放⼤大时,使⽤用线性过滤
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_LINEAR) 

glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_LINEAR);
```

#### 设置环绕方式
```
//参数1:GL_TEXTURE_1D、GL_TEXTURE_2D、GL_TEXTURE_3D 
//参数2:GL_TEXTURE_WRAP_S、GL_TEXTURE_T、GL_TEXTURE_R,针对s,t,r坐标 
//参数3:GL_REPEAT、GL_CLAMP、GL_CLAMP_TO_EDGE、GL_CLAMP_TO_BORDER
//GL_REPEAT:OpenGL 在纹理理坐标超过1.0的⽅方向上对纹理理进⾏行行重复; //GL_CLAMP:所需的纹理理单元取⾃自纹理理边界或TEXTURE_BORDER_COLOR. //GL_CLAMP_TO_EDGE环绕模式强制对范围之外的纹理理坐标沿着合法的纹理理单元的最后⼀⾏或者最后⼀列来进⾏行行采样。
//GL_CLAMP_TO_BORDER:在纹理理坐标在0.0到1.0范围之外的只使⽤用边界纹理理单元。边界纹理理单元是作为围绕基本图像的额外的⾏行行和列列，并与基本纹理理图像⼀一起加载的。glTextParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAR_S,GL_CLAMP_TO_EDGE);
glTextParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAR_T,GL_CLAMP_TO_EDGE);
```

![环绕方式](https://upload-images.jianshu.io/upload_images/1367029-c1a68a6f1c996588.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![环绕方式效果图](https://upload-images.jianshu.io/upload_images/1367029-7a833d2ed19df34f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 有关金字塔纹理坐标
![金字塔坐标](https://upload-images.jianshu.io/upload_images/1367029-553b7511ad118965.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

顶点坐标:
vBackLeft (-1.0,-1.0,-1.0)
vBackRight (1.0,-1.0,-1.0)
vFrontLeft (-1.0,-1.0,1.0)
vFrontRight(1.0,-1.0,1.0) 
vApex (0,1.0,0)

纹理理坐标
VBackLeft ( 0,0,0)
VBackRight (0,1,0)
vFrontLeft (0,0,1) 
VFrontRight (0,1,1) 
vApex (0,0.5,1)


