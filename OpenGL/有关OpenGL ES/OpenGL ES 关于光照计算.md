有关光照的代码公式, 在此用CC老师已经写好的代码做一个记录, 方便以后使用的时候查询.

记录一个函数-->根据你的设置返回一个4x4矩阵变换的世界坐标系坐标。

```
//获取世界坐标系去模型矩阵中.  
    /*  
     LKMatrix4 GLKMatrix4MakeLookAt(float eyeX, float eyeY, float eyeZ,  
     float centerX, float centerY, float centerZ,  
     float upX, float upY, float upZ)  
     等价于 OpenGL 中  
     void gluLookAt(GLdouble eyex,GLdouble eyey,GLdouble eyez,GLdouble centerx,GLdouble centery,GLdouble centerz,GLdouble upx,GLdouble upy,GLdouble upz);  
     
     目的:根据你的设置返回一个4x4矩阵变换的世界坐标系坐标。  
     参数1:眼睛位置的x坐标  
     参数2:眼睛位置的y坐标  
     参数3:眼睛位置的z坐标  
     第一组:就是脑袋的位置  
     
     参数4:正在观察的点的X坐标  
     参数5:正在观察的点的Y坐标  
     参数6:正在观察的点的Z坐标  
     第二组:就是眼睛所看物体的位置  
     
     参数7:摄像机上向量的x坐标  
     参数8:摄像机上向量的y坐标  
     参数9:摄像机上向量的z坐标  
     第三组:就是头顶朝向的方向(因为你可以头歪着的状态看物体)  
     */  
    
    GLKMatrix4 view1 = GLKMatrix4MakeLookAt(camX,camX,camZ,0,0,0,0,1,0);  
```

**其中注意 in out 相对应的输入输入属性写法, 其实与attribute varying的意思是一样的, 注意此处的写法**  
**以及在外部的使用与attribute varying的区别**

```
    glEnableVertexAttribArray(0);  
    glEnableVertexAttribArray(1);  
    glEnableVertexAttribArray(2);  
    
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 8*sizeof(float), (void*)NULL);  
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 8*sizeof(float), (void*)(3*sizeof(float)));  
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8*sizeof(float), (void*)(6*sizeof(float)));  
```

* 顶点着色器相应代码 (在此的作用 主要用来相应解释片元着色器代码)  

```
#version 300 es  

layout(location = 0) in vec3 position;  //顶点  
layout(location = 1) in vec3 normal;    //法向量  
layout(location = 2) in vec2 texCoord;  //纹理坐标  

uniform mat4 view;  
uniform mat4 projection;  

out vec3 outNormal; //法向量  
out vec3 FragPo;    //顶点在世界坐标位置  
out vec2 outTexCoord;//纹理坐标  

void main()  
{  

    FragPo = position;  
    outNormal = normal;  
    outTexCoord = texCoord;  
    gl_Position = projection * view * vec4(position,1.0);  
    
}  
```

* **重点查看片元着色器中 光线的计算公式**

```  
#version 300 es  

precision mediump float;  
out vec4 FragColor;  

uniform vec3 lightColor;    //光源颜色  
uniform vec3 lightPo;       //光源位置  
uniform vec3 viewPo;        //视角位置  
uniform sampler2D Texture;          //物体纹理  
uniform sampler2D specularTexture;  //镜面纹理  

in vec2 outTexCoord;    //纹理坐标  
in vec3 outNormal;      //顶点法向量  
in vec3 FragPo;         //顶点坐标  

//点光源版本
void pointLight(){
    
    float ambientStrength = 0.3;    //环境因子  
    float specularStrength = 2.0;   //镜面强度  
    float reflectance = 256.0;      //反射率  

    float constantPara = 1.0f;     //距离衰减常量  
    float linearPara = 0.09f;      //线性衰减常量  
    float quadraticPara = 0.032f;  //二次衰减常量  

    //环境光 = 环境因子 * 物体的材质颜色  
    vec3 ambient = ambientStrength * texture(Texture,outTexCoord).rgb;  

    //漫反射
    vec3 norm = normalize(outNormal);  
    //当前顶点 至 光源的的单位向量  
    vec3 lightDir = normalize(lightPo - FragPo);  
    //DiffuseFactor=光源与法线夹角 max(0,dot(N,L))  
    float diff = max(dot(norm,lightDir),0.0);  
    //漫反射光颜色计算 = 光源的漫反射颜色 * 物体的漫发射材质颜色 * DiffuseFactor  
    vec3 diffuse = diff * lightColor*texture(Texture,outTexCoord).rgb;  

    //镜面反射  
    vec3 viewDir = normalize(viewPo - FragPo);  
    // reflect (genType I, genType N),返回反射向量  
    vec3 reflectDir = reflect(-lightDir,outNormal);  
    //SpecularFactor = power(max(0,dot(N,H)),shininess)  
    float spec = pow(max(dot(viewDir, reflectDir),0.0),reflectance);  
    //镜面反射颜色 = 光源的镜面光的颜色 * 物体的镜面材质颜色 * SpecularFactor  
    vec3 specular = specularStrength * spec * texture(specularTexture,outTexCoord).rgb;  

    //衰减因子计算  
    float LFDistance = length(lightPo - FragPo);  
    //衰减因子 =  1.0/(距离衰减常量 + 线性衰减常量 * 距离 + 二次衰减常量 * 距离的平方)  
    float lightWeakPara = 1.0/(constantPara + linearPara * LFDistance + quadraticPara * (LFDistance*LFDistance));  
    
    //光照颜色 =(环境颜色 + 漫反射颜色 + 镜面反射颜色)* 衰减因子  
    vec3 res = (ambient + diffuse + specular)*lightWeakPara;  

    //最终输出的颜色  
    FragColor = vec4(res,1.0);  

}

// 平行光版本  
void parallelLight(){  
  
    float ambientStrength = 0.3;    //环境因子  
    float specularStrength = 2.0;   //镜面强度  
    float reflectance = 256.0;      //反射率  

    //平行光方向
    //vec3 paraLightDir = normalize(vec3(-0.2,-1.0,-0.3));  
    vec3 paraLightDir =normalize(vec3(-1,-1,1));  

    //环境光 = 环境因子 * 物体的材质颜色  
    vec3 ambient = ambientStrength * texture(Texture,outTexCoord).rgb;  

    //漫反射  
    vec3 norm = normalize(outNormal);  
    //当前顶点至光源的的单位向量  
    vec3 lightDir = normalize(lightPo - FragPo);  
    //DiffuseFactor=光源与paraLightDir 平行光夹角 max(0,dot(N,L))  
    float diff = max(dot(norm,paraLightDir),0.0);  
    //漫反射光颜色计算 = 光源的漫反射颜色 * 物体的漫发射材质颜色 * DiffuseFactor  
    vec3 diffuse = diff * lightColor * texture(Texture,outTexCoord).rgb;  

    //镜面反射  
    vec3 viewDir = normalize(viewPo - FragPo);  
    // reflect (genType I, genType N),返回反射向量 -paraLightDir平行光  
    vec3 reflectDir = reflect(-paraLightDir,outNormal);  
    //SpecularFactor = power(max(0,dot(N,H)),shininess)  
    float spec = pow(max(dot(viewDir, reflectDir),0.0),reflectance);  
    //镜面反射颜色 = 光源的镜面光的颜色 * 物体的镜面材质颜色 * SpecularFactor  
    vec3 specular = specularStrength * spec * texture(specularTexture,outTexCoord).rgb;  

    //距离衰减常量  
    float constantPara = 1.0f;  
    //线性衰减常量  
    float linearPara = 0.09f;  
    //二次衰减常量  
    float quadraticPara = 0.032f;  
    //衰减因子计算  
    float LFDistance = length(lightPo - FragPo);  
    //衰减因子 =  1.0/(距离衰减常量 + 线性衰减常量 * 距离 + 二次衰减常量 * 距离的平方)  
    float lightWeakPara = 1.0/(constantPara + linearPara * LFDistance + quadraticPara * (LFDistance*LFDistance));  

    //光照颜色 =(环境颜色 + 漫反射颜色 + 镜面反射颜色)* 衰减因子  
    vec3 res = (ambient + diffuse + specular)*lightWeakPara;  
    
    //最终输出的颜色  
    FragColor = vec4(res,1.0);  
}  

//聚光版本  
void Spotlight(){  
   
    float ambientStrength = 0.3;    //环境因子  
    float specularStrength = 2.0;   //镜面强度  
    float reflectance = 256.0;      //反射率  

    //环境光 = 环境因子 * 物体的材质颜色  
    vec3 ambient = ambientStrength * texture(Texture,outTexCoord).rgb;  

    //漫反射  
    vec3 norm = normalize(outNormal);  
    vec3 lightDir = normalize(lightPo - FragPo);    //当前顶点 至 光源的的单位向量  
    //DiffuseFactor=光源与paraLightDir lightDir夹角 max(0,dot(N,L))  
    float diff = max(dot(norm,lightDir),0.0);   //光源与法线夹角  
    //漫反射光颜色计算 = 光源的漫反射颜色 * 物体的漫发射材质颜色 * DiffuseFactor  
    vec3 diffuse = diff * lightColor*texture(Texture,outTexCoord).rgb;  

    //镜面反射  
    vec3 viewDir = normalize(viewPo - FragPo);  
     // reflect (genType I, genType N),返回反射向量    
    vec3 reflectDir = reflect(-lightDir,outNormal);  
    //SpecularFactor = power(max(0,dot(N,H)),shininess)  
    float spec = pow(max(dot(viewDir, reflectDir),0.0),reflectance);  
    //镜面反射颜色 = 光源的镜面光的颜色 * 物体的镜面材质颜色 * SpecularFactor  
    vec3 specular = specularStrength * spec * texture(specularTexture,outTexCoord).rgb;  

    float constantPara = 1.0f;    //距离衰减常量  
    float linearPara = 0.09f;     //线性衰减常量  
    float quadraticPara = 0.032f; //二次衰减常量  
    
    //衰减因子计算  
    float LFDistance = length(lightPo - FragPo);  
    //衰减因子 =  1.0/(距离衰减常量 + 线性衰减常量 * 距离 + 二次衰减常量 * 距离的平方)  
    float lightWeakPara = 1.0/(constantPara + linearPara * LFDistance + quadraticPara * (LFDistance*LFDistance));  

    //聚光灯切角 (一些复杂的计算操作 应该让CPU做,提高效率,不变的量也建议外部传输,避免重复计算)  
    float inCutOff = cos(radians(10.0f));  
    float outCutOff = cos(radians(15.0f));  
    vec3 spotDir = vec3(-1.2f,-1.0f,-2.0f);  
    
    //聚光灯因子 = clamp((外环的聚光灯角度cos值 - 当前顶点的聚光灯角度cos值)/(外环的聚光灯角度cos值- 内环聚光灯的角度的cos值),0,1)；  
    float theta = dot(lightDir,normalize(-spotDir));  
    //(外环的聚光灯角度cos值- 内环聚光灯的角度的cos值)  
    float epsilon  = inCutOff - outCutOff;  
    //(外环的聚光灯角度cos值 - 当前顶点的聚光灯角度cos值) / (外环的聚光灯角度cos值- 内环聚光灯的角度的cos值)  
    float intensity = clamp((theta - outCutOff)/epsilon,0.0,1.0);  
    vec3 res = (ambient + diffuse + specular)*intensity*lightWeakPara;  

    FragColor = vec4(res,1.0);  
}  

void DiffultLight(){  
    
    float ambientStrength = 0.3;    //环境因子  
    //环境光 = 环境因子 * 物体的材质颜色  
    vec3 ambient = ambientStrength * texture(Texture,outTexCoord).rgb;  
    
    //光源方向  
    //vec3 paraLightDir =normalize(vec3(0,1,0));  

    //漫反射  
    vec3 norm = normalize(outNormal);  
    vec3 lightDir = normalize(lightPo - FragPo);    //当前顶点 至 光源的的单位向量  
    //DiffuseFactor=光源与paraLightDir lightDir夹角 max(0,dot(N,L))  
    float diff = max(dot(norm,lightDir),0.0);   //光源与法线夹角  
    //漫反射光颜色计算 = 光源的漫反射颜色 * 物体的漫发射材质颜色 * DiffuseFactor  
    vec3 diffuse = diff * lightColor * texture(Texture,outTexCoord).rgb;  
    
     vec3 res = ambient + diffuse;  
     FragColor = vec4(res,1.0);  
}


void main()  
{
    //聚光版本  
    //Spotlight();  
    //点光源版本  
    //pointLight();  
    //平行光版本  
    //parallelLight();  
     DiffultLight();  
    
}  
```