因为着色器里面没有编译器提示, 所以熟悉着色器代码语法是非常有必要的

* 变量和数据类型

```
//布尔型. true/false
bool bDone = false;
//有符合整型数据
int iValue = 42;
//无符号整型数据
uint uiValue = 3929u;
//浮点型
float fValue = 3.14159f;
```

* 向量数据类型
*注意: 接下来就假设所有参与运算的变量已经赋值过了*

```
//1. 向量声明-- 4分量的float类型向量
vec4 V1;

//2. 声明向量并且对其进行构造
vec4 V2 = vec4(1,2,3,4);

//3. 向量运算, 加, 赋值给另外一个向量, 与标量相乘
vec4 v;
vec4 vOldPos = vec4(1,2,3,4);
vec4 vOffset = vec4(1,2,3,4);

  v = vOldPos + vOffset;
  v = vNewPos;
  v += vec4(10,10,10,10);
  v = vOldPos * vOffset;
  v *= 5;

//4. 向量中元素的获取, 可以通过x, y, z, w来获取向量中的元素值
 􏰧􏰮􏲐􏲑x,y,z,w􏴯􏳢􏳣􏲼􏲮􏲼􏲮􏰊􏰦􏲢􏳥􏳟 v1.x = 3.0f;
 v1.xy = vec2(3.0f,4.0f);
 v1.xyz = vec3(3,0f,4,0f,5.0f);
 
//5. 可以通过颜色控制r, g, b, a
v1.r = 3.0f;
v1.rgba = vec4(1.0f,1.0f,1.0f,1.0f);

//6. 可以通过纹理坐标stpq
v1.st = vec2(1.0f,0.0f);

//7.􏵠􏵡 注意! 赋值混合不合法 不可以
v1.st = v2.xt; //􏳂􏰧􏰮不可以
v1.st = v2.xy; //􏰧􏰮可以, 没有开发意义, 会被喷死

//8. 向量支持调换(swizzle)操作, 2个或2个以上向量元素来进行交换
v1.rgba = v2.bgra;
v2.bgra = v1.rgba;
//􏳞􏳟􏰠􏰡 赋值操作
v1.r = v2.b; 
v1.g = v2.g; 
v1.b = v2.r; 
v1.a = v2.a;

//9.􏲼􏲮􏵮􏵪􏵫􏰂􏱜􏵯􏰍􏲆􏰏􏵘􏲮􏰠􏰡 向量还支持一次性对所有分量操作
v1.x = vOtherVerex.x +5.0f; 
v1.y = vOtherVerex.y +4.0f; 
v1.z = vOtherVerex.z +3.0f;
v1.xyz = vOtherVerex.xyz + vec3(5.0f,4.0f,3.0f);

```

* 矩阵

```
//创建矩阵
mat4 m1,m2,m3;
//2.􏴪􏴫􏵰􏲢􏲽􏲾 构造单元矩阵
mat4 m2 = mat4(1.0f,0.0f,0.0f,0.0f
                    0.0f,1.0f,0.0f,0.0f,
                    0.0f,0.0f,1.0f,0.0f,
                    0.0f,0.0f,0.0f,1.0f);
mat4 m4 = mat4(1.0f);
mat4 m3 = mat4(0.5,0.5,0.5,0.5,
				0.5,0.5,0.5,0.5,
				0.5,0.5,0.5,0.5,
				0.5,0.5,0.5,0.5,)
				
m1 = m2 * m3;
```

* const

```
const  float zero = 0.0;
```

* 结构体

```
 struct forStruct{
 vec4 color;
 float start;
 float end;
}fogVar;

fogVar = fogStruct(vec4(1.0,0.0,0.0,1.0),0.5,2.0);
vec4 color = fogVar.color;
float start = fogVar.start;
```

* 数组

```
float floatArray[4];
vec4 vecArray[2];
//注意
float a[4] = float[](1.0,2.0,3.0,4.0);
vec2 c[2] = vec2[2](vec2(1.0,2.0),vec2(3.0,4.0));
```

* 函数
**注意: GLSL 函数没有递归**
	定义函数给3个修饰符
	* in: (没有指定时, 默认限定修饰符), 传递进入函数中, 函数不能对其进行修改.
	* inout: 传入相应数值, 并且可以在函数中进行修改
	* out: 函数返回时, 可以将其修改

```
vec4 myFunc(inout float myFloat, out vec4 m1, mat4 m2){

}
vec4 diffuse(vec3 normal ,vec3 light, vec4 baseColor) {
	return baseColor * dot(normal,light); 
}

```

* 控制语句

**循环只支持while 循环/do...while.../for**
**但是! OpenGL ES开发中, 尽量减轻逻辑判断, 尽量降低循环迭代使用.**

```
if(color.a < 0.25)
  {
      color *= color.a;
  }
  else
  {
      color = vec4(1.0,1.0,1.0,1.0);
}

```


