### 1. 在Xcode内新建一个项目
![image.png](https://upload-images.jianshu.io/upload_images/1367029-de63a07112d0bc3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2. 选macOS > APP
![image.png](https://upload-images.jianshu.io/upload_images/1367029-c11d255b9a3a3b3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3. 修改项目名
![image.png](https://upload-images.jianshu.io/upload_images/1367029-63f3c322fcc0d24d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4. 添加OpenGl.framework和GLUT.framework两个依赖库 
![image.png](https://upload-images.jianshu.io/upload_images/1367029-6a5fc7eb92e2c8e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 并删除文件AppDelegate.h AppDelegate.m main.m ViewController.h ViewController.m跟OC有关文件
![image.png](https://upload-images.jianshu.io/upload_images/1367029-80dd19284146143a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 5. 需要的文件include文件包和libGLToos.a
[百度网盘的分享](https://pan.baidu.com/s/19E-tDnP9ZuUwxKQsQJf2Zg)
提取码: b25e
![image.png](https://upload-images.jianshu.io/upload_images/1367029-00457d26b2ae3698.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 将文件拖入工程
![image.png](https://upload-images.jianshu.io/upload_images/1367029-2c6ddd4ca9d9e93d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 6. 新建C++文件 起名为main

![image.png](https://upload-images.jianshu.io/upload_images/1367029-d7a08a933113f2ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/1367029-5d01cbaad3aae46e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 7. signing & Capabilities 中证书改为 Sing to Run Locally
![image.png](https://upload-images.jianshu.io/upload_images/1367029-18604e588ed4034d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 8. 记得在Build Settings 中的 Header Search Paths中增加include文件夹的路径
![image.png](https://upload-images.jianshu.io/upload_images/1367029-9696634aacaba323.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 9. 加入测试代码 编辑运行
```
#include "GLShaderManager.h"
#include "GLTools.h"
#include <glut/glut.h>

GLBatch triangleBatch;
GLShaderManager shaderManager;

//窗口大小改变时接受新的宽度和高度，其中0,0代表窗口中视口的左下角坐标，w，h代表像素
void ChangeSize(int w,int h)
{
    glViewport(0,0, w, h);
}

//为程序作一次性的设置
void SetupRC()
{
    //设置背影颜色
    glClearColor(0.0f,0.0f,1.0f,1.0f);
    
    //初始化着色管理器
    shaderManager.InitializeStockShaders();
    
    //设置三角形，其中数组vVert包含所有3个顶点的x,y,笛卡尔坐标对。
    GLfloat vVerts[] = {
        -0.5f,0.0f,0.0f,
        0.5f,0.0f,0.0f,
        0.0f,0.5f,0.0f,
    };
    
    //批次处理
    triangleBatch.Begin(GL_TRIANGLES,3);
    triangleBatch.CopyVertexData3f(vVerts);
    triangleBatch.End();
}

//开始渲染
void RenderScene(void)
{
    //清除一个或一组特定的缓冲区
    glClear(GL_COLOR_BUFFER_BIT|GL_DEPTH_BUFFER_BIT|GL_STENCIL_BUFFER_BIT);
    
    //设置一组浮点数来表示红色
    GLfloat vRed[] = {1.0f,0.0f,0.0f,1.0f};
    
    //传递到存储着色器，即GLT_SHADER_IDENTITY着色器，这个着色器只是使用指定颜色以默认笛卡尔坐标第在屏幕上渲染几何图形
    shaderManager.UseStockShader(GLT_SHADER_IDENTITY,vRed);
    
    //提交着色器
    triangleBatch.Draw();
    
    //将在后台缓冲区进行渲染，然后在结束时交换到前台
    glutSwapBuffers();
}

int main(int argc,char* argv[])
{
    //设置当前工作目录，针对MAC OS X
    gltSetWorkingDirectory(argv[0]);
    
    //初始化GLUT库
    glutInit(&argc, argv);
    
    /*初始化双缓冲窗口，其中标志GLUT_DOUBLE、GLUT_RGBA、GLUT_DEPTH、GLUT_STENCIL分别指
     
     双缓冲窗口、RGBA颜色模式、深度测试、模板缓冲区*/
    glutInitDisplayMode(GLUT_DOUBLE|GLUT_RGBA|GLUT_DEPTH|GLUT_STENCIL);
    
    //GLUT窗口大小，标题窗口
    glutInitWindowSize(800,600);
    
    glutCreateWindow("Triangle");
    
    //注册回调函数
    glutReshapeFunc(ChangeSize);
    
    glutDisplayFunc(RenderScene);
    
    //驱动程序的初始化中没有出现任何问题。
    GLenum err = glewInit();
    if(GLEW_OK != err) {
        fprintf(stderr,"glew error:%s\n",glewGetErrorString(err));
        return 1;
    }
    
    //调用SetupRC
    SetupRC();
    glutMainLoop();
    return 0;
}

```