![精度说明表](https://upload-images.jianshu.io/upload_images/1367029-4d91175e4f0cef88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Satisfies the minimum requirements for the vertex language described above.Optional in the fragment language

满足上面描述的顶点语言的最低要求, 在片段语言中是可选的

satisfies the minimum requirements above for the fragment language . Its range and precision has to be greater than or the same as provided by lowp and less than of the same as provided by highp.

满足上面片段语言的最低要求. 其范围和精度必须大于或等于lowp提供的范围和精度, 小于highp提供的范围和精度

Range and precision that can be less than mediump , but still intended to represent all color values for any color channel.

范围和精度可以小于mediump, 但仍用于表示任何颜色通道的所有颜色值.

例如: 

```
lowp float color;

varying mediump vec2 Coord; 

highp mat4 m;
```

#### 精度范围
![精度范围](https://upload-images.jianshu.io/upload_images/1367029-7a21e684025e3834.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于高精度和中精度, 整型范围必须可以准确地转化成相应的相同精度修饰符所表示的float型.

例如: highp int 可以被转换成 highp float, mediump int 可以被转化为 mediump float, 但是lowp int 不能转化为lowp float.

字符常量和布尔型没有精度修饰符, 当浮点数和整数构造器不含带有精度修饰的参数时也不需要精度修饰符.

1. 指定变量精度放在数据类型之前 

```
highp vec4 position; 

varying lowp vec4 color; 

mediump float specularExp;
```

2. 指定默认精度放在Vertex和Fragment shader源码开始处

precision precision-qualifier type;

* precision可以用来确认默认精度修饰符
* precision-qualifier可以是lowp, mediump或者highp. 任何其他类型和修饰符都会引起错误.
* 如果type是float类型, 那么该精度(precision-qualifier)将适用于所有无精度修饰符的浮点数声明 (标量, 向量, 矩阵);
* 如果type是int类型, 那么改精度(precision-qualifier)将适用于所有无精度修饰符的整数声明(标量, 向量);
* 没有声明精度修饰符的变量将使用和它最近的precision语句中的精度.

例如: 
precision highp float;
precision mediump int;

**注意**

**在Vertex shader中, 如果没有默认的精度, 则float和int精度都为highp:**
**在Fragment shader中, float没有默认的精度, 所以必须在Fragment shader中为float指定一个默认精度或为每个float变量指定精度**

3. 预定义精度

在顶点语言中有如下预定义的全局默认精度语句

```
precision highp float; 

precision highp int; 

precision lowp sampler2D; 

precision lowp samplerCube;
```

在片元语言中有如下预定义的全局默认精度语句:

```
precision mediump int; 

precision lowp sampler2D; 

precision lowp samplerCube;
```

片元语言没有默认的浮点数精度修饰符. 因此, 对于浮点数, 浮点数向量和矩阵变量声明, 要么声明必须包含一个精度修饰符, 要不默认的精度修饰符在之前已经被声明过了.


