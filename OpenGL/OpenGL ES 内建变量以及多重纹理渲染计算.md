我们用GLSL来编写着色器代码时, 要了解他们的一些内建变量参数的含义. 本文就是对内建变量做一些介绍.  

### 在顶点着色器中的内建变量

**顶点着色器的内建变量可以分为特殊变量(顶点着色器输入/输出), 统一状态(深度范围)以及规定的最大值(属性数量, 顶点着色器变量数量和统一变量数量)的常量.**

* gl_VertexID: 是一个输入变量, 用于保存顶点的整数索引. 这个整数型变量, 用highp精度限定字符声明.  
* gl_Instance: 是一个输入变量, 用于保存实例化绘图调用中图元的实例编号, 对于常规调用的绘图调用, 该值为0; gl_InstanceID是一个整数型常量, 用highp精度限定字符声明.
* gl_Position: 用于输出顶点位置的剪裁坐标, 该值在剪裁和视口变换用于执行相应的图元裁剪以及从裁剪坐标到屏幕坐标的顶点位置转换, 如果顶点着色器未写入gl_Position, 则gl_Position的值未定义, gl_Positon是浮点变量, 用highp精度限定字符声明.
* gl_PointSize: 可以写入像素表示点精灵的尺寸, 在渲染点精灵时使用. 顶点着色器的输出的gl_PointSize值被限定在OpenGL ES 3.0实现支持的非平滑点大小范围之内. gl_PointSize是一个浮点变量, 用highp精度限定字符声明.
* gl_FrontFacing: 是一个特殊变量, 但不是由顶点着色器直接写入的, 二是根据顶点着色器生成的位置值和渲染图元的类型生成的, 它是一个布尔值.

**在顶点着色器内可用的唯一内建 Uniform状态是窗口坐标中的深度范围, 这些由内建统一变量名 gl_DepthRange 给出.**

```
struct gl_DepthRangeParameters {  
highp float near; //near z  
highp float far; //near far  
highp float diff; //far - near  
}  
uniform gl_DepthRangeParameters gl_DepthRange;  
```

**顶点着色器中的内建常量**

```
const mediump int gl_MaxVertexAttribs = 16;  
const mediump int gl_MaxVertexUniformVectors = 256;  
const mediump int gl_MaxVertexOutputVectors = 16;  
const mediump int gl_MaxVertexTextureImageUnits = 16;   
const mediump int gl_MaxCombinedTextureImageUnits = 32;  
```
gl_MaxVertexAttribs : 可以指定的顶点数据最大的数量.  
gl_MaxVertexUniformVectors : 顶点着色器可以使用的vec4 Uniform 变量最大数量.  
gl_MaxVertexOutputVectors : 是输出向量的足底啊数量.  
gl_MaxVertexTextureImageUnits : 顶点着色器可用纹理单元的最大数量.  
gl_MaxCombinedTextureImageUnits : 顶点/片元着色器中的可用纹理单元的最大数量的总和.   

### 片元着色器的内建特殊变量

* gl_FragCoord : 片段着色器中一个只读变量, 这个变量保存片段的窗口相对坐标.  
* gl_FrontFacing : 片段着色器中的一个只读变量, 这个布尔变量时正面图元是位true, 否则为false;  
* gl_FragDepth : 一个只写输出变量, 在片段着色器写入时, 覆盖片段的固定功能深度值. 尽量减少手动实现深度值写入. 这个功能需要谨慎使用 因为它可能禁用许多GPU的深度优化, 例如, 许多GPU都有"Early -Z"功能, 在执行片段着色器之前进行深度测试, 使用"Early -Z"的好处就是不能通过深度测试的片段就不会被着色器(从而减低了着色器调用次数, 提高了性能), 但是使用gl_FragDepth, 就必须禁用该功能, 因为GPU在执行着色器之前不知道深度值. 

**片元着色器中的内建常量**

```
const mediump int gl_MaxFragmentInputVectors = 15;   
const mediump int gl_MaxTextureImageUnits = 16;   
const mediump int gl_MaxFragmentUniformVectors = 224;   
onst mediump int gl_MaxDrawBuffers = 4;
const mediump int gl_MinProgramTexelOffset = -8;   
const mediump int gl_MaxProgramTexelOffset = 7;  
```
gl_MaxFragmentInputVectors : 片段着色器输入的最大数量.  
gl_MaxTextureImageUnits : 可用纹理图像单元的最大数量.  
gl_MaxFragmentUniformVectors : 片段着色器可用vec4 Uniform变量最大数量.  
gl_MaxDrawBuffers : 多重渲染目标最大支持数量.  

### 多重纹理渲染计算

意思就是最终颜色, 由多张纹理合成计算.  

```
attribute vec2 v_texCoord;   
uniform sampler2D s_baseMap;   
uniform sampler2D s_SecondMap;  
 void main()  
 {  
 	vec4 baseColor;  
 	vec4 secondColor;  
 	baseColor = texture(s_baseMap, v_texCoord);  
 	secondColor = texture(s_SecondMap, v_texCoord);  
 	gl_FragColor = baseColor * secondColor;  
 }  
```

客户端代码: 将各个纹理对象绑定到纹理单元0和1, 为采样设置数值, 将采集器绑定到对应的纹理单元  

```
  glActiveTexutre(GL_TEXTURE0);   
  glBindTeture(GL_TEXTURE_2D,baseMapTexId);     
  glUniformli(baseMapTexId,0);    

    
  glActiveTexutre(GL_TEXTURE1);  
  glBindTeture(GL_TEXTURE_2D,secondMapTexId);  
  glUniformli(secondMapTexId,1);  
```



