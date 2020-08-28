##### 压栈 PushMatrix();
```
modelViewMatrix.PushMatrix();
```
这句代码的意思是压栈，如果 PushMatix() 括号里是空的，就代表是把栈顶的矩阵复制一份，再压栈到它的顶部。如果不是空的，比如是括号里是单元矩阵，那么就代表压入一个单元矩阵到栈顶了。

##### 矩阵相乘 MultMatrix(mObjectFrame)
```
// 将modelViewMatrix 的堆栈中的矩阵 与 mOjbectFrame 矩阵相乘，存储到modelViewMatrix矩阵堆栈中
modelViewMatrix.MultMatrix(mObjectFrame);
```
这句代码的意思是把 模型视图矩阵堆栈 的 栈顶 的矩阵copy出一份来和新矩阵进行矩阵相乘，然后再将相乘的结果赋值给栈顶的矩阵。
下图是该函数的系统声明:

![MultMatrix](https://upload-images.jianshu.io/upload_images/1802326-3a3d765b4fd614ff.png?imageMogr2/auto-orient/strip|imageView2/2/w/1008/format/webp)

##### 出栈 PopMatrix();
```
modelViewMatrix.PopMatrix();
```
把栈顶的矩阵出栈，恢复为原始的矩阵堆栈，这样就不会影响后续的操作了。


下面是引用一位学长所绘制的压栈出栈图, 一目了然:

![矩阵入栈相乘出栈](https://upload-images.jianshu.io/upload_images/1802326-00480c3517d10f09.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)