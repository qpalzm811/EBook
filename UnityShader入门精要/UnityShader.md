[高清彩图](https://blog.csdn.net/Raymond_King123/article/details/115661077)

# 第三章 Unityshader基础

## 3.1 unity shader概述

### 3.1.3 unity 中的shader

unity shader与之前提及的渲染管线的shader很大不同。

Unity提供了四种模板

1. Standard Surface Shader：产生一个包含 标准光照模型，使用unity5新加的基于物理的渲染方法 的 表面着色器模板。
2. Unlit Shader：产生不包含 光照 （但包含雾效）的基本 顶点 片元着色器
3. Image Effect Shader：实现各种屏幕后处理效果 基本模板
4. Computer Shader：特殊shader，用GPU进行常规渲染流水线无关的计算（本书不讨论）

Unity Shader必须和材质Material结合起来

## 3.2 Unity Shader的基础ShaderLab
- ShaderLab是Unity提供的Unity Shader说明性语言
- 使用嵌套在花括号内部的语义（Syntax）没教书Unity Shader文件的结构
- 会根据使用的平台将这些结构，编译成真正的代码和Shader文件。

```
Shader "ShaderName"{
	Properties{
		//属性
	}
	SubShader{
		//显卡A使用的子着色器
	}
	SubShader{
		//显卡A使用的子着色器
	}
}
```

## 3.3 UnityShader的结构

### 3.3.2 材质 Material 和 UnityShader的桥梁：Properties

[ShaderLab: defining material properties](https://docs.unity3d.com/Manual/SL-Properties.html)

|     **Type**     | **Example syntax**                                           | **Comment**                                                  |
| :--------------: | :----------------------------------------------------------- | :----------------------------------------------------------- |
|   **Integer**    | `_ExampleName ("Integer display name", Integer) = 1`         | 真正的整形Integer，不像Int是用浮点实现的。<br>为了兼容性可以使用Int |
| **Int** (legacy) | `_ExampleName ("Int display name", Int) = 1`                 | 只是为了兼容性，浮点实现。若整形用**Integer**                |
|    **Float**     | ` _ExampleName ("Float display name", Float) = 0.5`<br> `_ExampleName ("Float with range", Range(0.0, 1.0)) = 0.5` | 浮点，Range可以表示一个拉动条                                |
|      **2D**      | `_ExampleName ("Texture2D display name", 2D) = "" {}` <br> `_ExampleName ("Texture2D display name", 2D) = "red" {}` | 下值放在字符串中使用unity内置纹理<br>“white” (RGBA: 1,1,1,1),<br> “black” (RGBA: 0,0,0,1),<br> “gray” (RGBA: 0.5,0.5,0.5,1),<br> “bump” (RGBA: 0.5,0.5,1,0.5) ,<br> “red” (RGBA: 1,0,0,1).<br>不写默认“gray”.  **Note:** 这默认纹理在 Inspector不可见 |
|   **2DArray**    | `_ExampleName ("Texture2DArray display name", 2DArray) = "" {}` | For more information, see [Texture arrays](https://docs.unity3d.com/Manual/class-Texture2DArray.html). |
|      **3D**      | `_ExampleName ("Texture3D", 3D) = "" {}`                     | 默认“gray” (RGBA: 0.5,0.5,0.5,1) texture.                    |
|     **Cube**     | `_ExampleName ("Cubemap", Cube) = "" {}`                     | 默认“gray” (RGBA: 0.5,0.5,0.5,1) texture.                    |
| **CubemapArray** | `_ExampleName ("CubemapArray", CubeArray) = "" {}`           | See [Cubemap arrays](https://docs.unity3d.com/Manual/class-CubemapArray.html). |
|    **Color**     | `_ExampleName("Example color", Color) = (.25, .5, .5, 1)`    | 颜色选择器float4类型<br> 若想编辑四个单独浮点数，可用Vector类型 |
|    **Vector**    | `_ExampleName ("Example vector", Vector) = (.25, .5, .5, 1)` | float4类型，若想用颜色用Color                                |

- Int Float Integer Range 都是数字类型，Color，Vector 默认值用 ( ) 中的四维向量
- 2D, 3D, Cube是三种纹理值，用一个 字符串 和后面的 花括号 制定， 一般为内置的纹理名称"red"等
  - 花括号用于 指定**纹理属性**， unity5.0后需要自己在 顶点着色器 编写计算相关 **纹理坐标** 的 代码

![image-20220429085336547](UnityShader.assets/image-20220429085336547.png)

Properties 是为了让这些属性出现在材质面板中（补图）

### 3.3.3 重量级成员：SubShader

- 每个UnityShader文件可以包含多个SubShader语义块，至少一个。
- 需要加载的时候扫描每个SubShader 选择第一个可在 **目标平台** 上运行的SubShader。根据不同显卡进行不同的复杂度计算
- 都不支持的话，Unity使用Fallback语义指定的UnityShader
```c#
SubShader{
	//可选的
	[Tags]
	[RenderSetup]
	Pass{	}
}
```
1. SubShader定义了一系列Pass 和 可选的 **状态**`[RenderSetup]`和 **标签**`[Tags]`
2. 每个Pass定义了一次完整的渲染流程，过多的Pass造成渲染性能下降
3. RenderSetup 和 Tags同样可以在 Pass中声明，在RenderSetup和Tags中设置的可用于所有Pass

 #### 状态设置 RenderSetup

渲染状态设置指令，可以设置显卡各种状态

![image-20220427104623531](UnityShader.assets/image-20220427104623531.png)

若写在RenderSetup中，则应用到所有的Pass，若不想如此，可以在**Pass中单独设置**

#### 标签 Tags

- Tags是一个键值对，key和value都是字符串string类型
- 告诉渲染引擎 希望 **怎样 何时** 渲染这个对象
- 类似：`Tags{ "TagName1" = "Value1" }`

![image-20220427112009975](UnityShader.assets/image-20220427112009975.png)

- **上述Tags 仅可以在Sunshader声明， 不可以在Pass中声明！**
- **Pass中也可以声明标签，但是不同于SubShader的标签类型**

#### Pass
```
Pass{
	[Name]
	[Tags]
	[RenderSetup]
	// other code
}
```
1. 先在Pass中定义 Pass 的名称 `Name “MyPassName”`：
   1. 通过Name，可以直接用其他的Shader的Passs，`UsePass “MyShader/MYPASSNAME”`
   2. **Unity会把所有的Pass名称转为大写，一定要用大写**
2. Pass可以设置渲染状态 **RenderSetup**,除了上面的状态设置外,还可以用 固定管线的着色器(3.4.3)命令
3. Pass还可以设置标签**Tags**，不同于SubShader的标签。 但也是用于告诉渲染引擎，怎样渲染该物体的
   1. ![image-20220427154643950](UnityShader.assets/image-20220427154643950.png)

4. UnityShader还支持特殊Pass，用于代码复用或者更复杂效果
   1. **UsePass**：复用其他Shader 的Pass
   2. **GrabPass**：改Pass负责抓取屏幕，并将结果存储在一张纹理中，后续的Pass处理（详见10.2.2）


### 3.3.4 后路：Fallback

- 一个Shader的所有SubShader都运行不了，就用这个 Fallback
- `Fallback “name"`或`Fallbcak Off`
- Fallback还会影响阴影的投射，
  - 在**渲染阴影纹理**的时候，unity会在每个Shader中找到一个阴影投射的Pass
  - 一般，我们不需要 专门实现一个Pass，因为Fallback包含了一个通用的Pass
  - 因此为每个Shader设置正确的Fallback很重要，详见9.4节

自定义材质面板的编辑界面，可以使用CustomEditor扩展编辑界面

## 3.4 Unity Shader的形式

- UnityShader最主要的工作还是指定**各种着色器**的代码。
- 可以写在 SubShader中（表面着色器）
- 可以写在Pass 块中（顶点/片元着色器 和 固定函数着色器做法）

###  3.4.1Unity宠儿 ：表面着色器 SurfaceShader

1. 这是Unity自造的着色器，可以当做一种 顶点/片元着色器的高层抽象
2. 为我们处理了很多光照细节

```C#
Shader "Custom/Simple Surface Shader"{
	SunShader {
		Tags{ "RenderType" = "Opaque"}
		CGPROGRAM
		#pragma surface surf Lambert
		struct Input {
			float4 color : COLOR;
		};
		void surf (Input IN, inout SurfaceOutput o){
			o.Albedo = 1;
		}
		ENDCG
	}
	Fallback "Diffuse"
}
```

- 表面着色器定义在SubShader 的 CGPROGRAM和 ENDCG之间
- 因为，Surface Shader不需要关心用多少个Pass，每个Pass如何渲染，Unity为我们做好
- CGPROGRAM 和 ENDCG 之间的代码是使用CG/HLSL编写的

### 3.4.2 最聪明的孩子：顶点/片元着色器

Unity中可以使用CG/HLSL语言编写顶点/片元着色器（Vertex/Fragment Shader)

```C#
Shader "Custom/Simple Surface Shader"{
	SunShader {
        	Pass{
                	CDPROGRAM
                        #pragma vertex vert
                        #pragma fragment frag
                        float4 vert(float4 v : POSITON) : SV_POSITION{
                        	return mul {UNITY_MATRIX_MVP, v};
                        }
                	fixed4 frag() : SV_Target {
                        	return fixed4(1.0, 0.0, 0.0, 1.0);
                        }
                        ENDCG
            }
	}
}
```

1. Surface Shder一样，Vertex/Fragment 也要定义在`CGPROGRAM` 和 `ENDCG`之间
2. 不同：Vertex/Fragment**要写在Pass**语义块中，而不是SubShader中

### 3.4.3 被抛弃的角落： 固定函数着色器 Fixed Function Shader

对于老旧设备，GPU仅支持DirectX7、OpenGL 1.5 或者OpenGL ES 1.5，例如iPhone3**不支持可编程管线Shader**，需要用到

```C#
Shader  "Tutortial/Basic"{
	Properties{
		_Color {"Main Color". Color} = {1, 0.5, 0.5, 1}
	}
	SubShader{
		Pass{
			Material{
				Diffuse { _Color }
			}
			Lighting On
		}
	}
}
```

Fixed Function Shader定义在 Pass 语义块，相当于一些渲染设置，我们需要完全使用ShaderLab语法编写

### 3.4.4 选择UnityShader形式的建议

1. 如果是 **光源**打交道，推荐表面着色器 Surface Shader
2. 如果是 光源少，例如只有一个平行光，推荐 顶点/片元着色器Vertex/Fragment Shader
3. 如果有更多 **自定义效果**，推荐使用 顶点/片元着色器Vertex/Fragment Shader

## 3.6 答疑解惑

### 3.6.1 UnityShader != 真正Shader 

UnityShader（ShaderLab）我们可以做到的远多于 一个 传统Shader

|                          传统Shader                          |                    UnityShader(ShaderLab)                    |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|         仅可编写特定Shader，如顶点着色器，片元着色器         |           一个文件中同时包含顶点着色器，片元着色器           |
| 无法设置一些渲染设置，如是否开启混合，深度测试等<br>是另外代码设置的 |                   通过一行特定指令即可设置                   |
|    冗长代码设置**输入输出**，小心处理输入输入位置对应关系    | 只需要特定语句块声明一些属性，便可依靠材质方便改变这些属性<br>对于模型自带的数据（顶点位置，纹理坐标，法线的等）提供直接访问的方法 |

### 3.6.2 UnityShader和CG/HLSL关系

- 通常CG代码是位于Pass语义块中的

- 但是，之前说讲 **表面着色器SurfaceShader**的时候，说CG是写在**SubShader**中的

  - 因为 表面着色器 会转化成 包含多个Pass的 顶点/片元着色器，以用于多种设备
  - 我们可在导入设置面板上点击 *Show Generated Code*看到生成的真正的 顶点/片元着色器 代码
  - 本质上，UnityShader只有两种形式，<u>**顶点/片元着色器**</u> 和 固定函数着色器(5.2后也会被转化为前者)

  # 第四章 Shader所需的数学模型

  ## 4.2笛卡尔坐标系

  ![image-20220429085541418](UnityShader.assets/image-20220429085541418.png)
  
  二维笛卡尔坐标系之间转换：1. 顺时针转180°，此时y轴方向一致，x轴方向相反。2.

1. 三维笛卡尔坐标系定义了3个坐标轴 和 1个原点
   1. 3个坐标轴是 **基矢量（basis vector）**，相互垂直，长度为 1
   2. 这种 **基矢量**被称为 **标准正交基（orthonormal basis）**
   3. 在一些坐标系中，坐标系之间相互垂直但是 长度不是 1 ，这种基矢量被称为 **正交基（orthogonal basis）**
   4. **正交**：相互垂直
2. 因为坐标轴的方向不固定，所以导致了两种不同的坐标系：**左手坐标系（left-handed coordinate space）**和**右手坐标系(right-handed coordinate space)**

### 4.2.3 左手坐标系和右手坐标系

- 对三维笛卡尔坐标系，靠二维的顺时针180°后水平翻转，不能让坐标系重合
- 三维坐标系，我们令两个坐标轴重合，第三条指向总是相反的。这种的就是不同的**旋向性（handedness）**

<img src="UnityShader.assets/image-20220507105135496.png" alt="image-20220507105135496" style="zoom:80%;" />

- **旋转的正方向：**用左手右手定则，确定旋转的正方向，大拇指指向旋转的轴正方向，四肢弯曲为正方向
  - ![image-20220507112246813](UnityShader.assets/image-20220507112246813.png)

- ![image-20220507141216930](UnityShader.assets/image-20220507141216930.png)

  - 相同的视觉现象，数学描述却是不同的

  - | 左手坐标 | X轴 +1 | Z轴 -4 | 旋转60°   |
    | -------- | ------ | ------ | --------- |
    | 右手坐标 | X轴 +1 | Z轴 +4 | 旋转-60度 |

  -  大拇指指向Y轴，握拳看旋转正反

### 4.2.4 Unity使用的坐标系

1. Unity使用的是**左手坐标系（left-handed  coordinate space）**
2. 但是观察空间，用到是**右手坐标系（right-handed coordinate space）**，摄像机前面方向是 **-Z轴**，与模型空间和世界空间相反，Z轴减少意味着场景深度增加（看绝对值就行）
3. <img src="UnityShader.assets/image-20220507172120210.png" alt="image-20220507172120210" style="zoom:100%;" />

### 4.2.5练习

1. 3dMax的：X正指向右，Y指向前，Z指向上，什么坐标系？
   1. 右手坐标系，大拇指x，食指y，中指z。从x握拳到z，大拇指指y轴
2. 左手坐标西，有一点(0,0,1)，绕y轴正方向旋转+90°，坐标多少？右手呢
   1. ![image-20220509094553496](UnityShader.assets/image-20220509094553496.png)
   2. (-1, 0, 0)
3. ![image-20220509095317761](UnityShader.assets/image-20220509095317761.png)
   1. 摄像机观察空间中，球体Z值是：10
   2. 摄像机模型空间下，球体Z值是：-10

## 4.3 点和矢量

- **点(Point)**，N维度空间中的一个位置，没有大小，宽度概念。
- **矢量(vector)**，为了和**标量(scalar)**区分，指<u>n维空间中包含**模(magnitude)**和**方向(direction)**的有向线段</u>。
  - 矢量通常被用于表示一个点的**偏移(displacement)**，是**相对量**，
  - 所以<u>**模和方向不变，无论放在哪里，都是同一个矢量**</u>

### 4.3.2 矢量运算

#### 1. **矢量标量乘除**

1. ![image-20220509105239590](UnityShader.assets/image-20220509105239590.png)

#### 2. **矢量加减法**![image-20220509113209008](UnityShader.assets/image-20220509113209008.png)

   1. ![image-20220509113350752](UnityShader.assets/image-20220509113350752.png)
   2. 如果我们想要计算**点b对点a**的位移，就可以通过**b-a**得到

#### 3. **矢量的模**![image-20220509133428905](UnityShader.assets/image-20220509133428905.png)

#### 4. **单位矢量（unit vector）**：模为1的矢量，表示我们只关心**方向（direction）**

   - 例如计算光照模型是，得到顶点的法线方向和光源方向

   - 单位向量也被称为**归一化向量（normalized vector）**，对于非零向量，转化为单位向量的过程就叫归一化

   - **归一化**![image-20220509135113959](UnityShader.assets/image-20220509135113959.png)
   - 零向量：v=(0,0,0)，不可以被归一化
   - 几何意义：![image-20220509140325886](UnityShader.assets/image-20220509140325886.png)
   - 当我们遇到**法线方向**，**光源方向**等，这些不一定是单位向量，在使用前应当**归一化运算**

#### 5. **矢量的点积（dot product）内积（inner product）**

   1. $ a · b = (a_x, a_y, a_z) · (b_x, b_y, b_z) = a_x b_x+ a_y b_y+ a_z b_z $

   2. 点积**几何意义**：**投影（projection）**

   3. ![image-20220509145837083](UnityShader.assets/image-20220509145837083.png)

   4. 点积结果的符号让我们知道两个矢量的方向关系：计算两个向量的夹角

   5. 性质

      1. ![image-20220509162618940](UnityShader.assets/image-20220509162618940.png)
      2. 从性质三，**我们可以用叉积计算<u>模</u>**，一般，我们只想比较**<u>模</u>**大小，就不必开平方消耗性能了

   6. 点积公式二：从三角代数出发，更具有几何意义

      1. $ 公式： a · b = |a||b| \cos \theta  $
      1. 从单位向量 $\hat{a} · \hat{b}  = \cos\theta= \frac{直角边}{斜边}$ 
      1. <img src="UnityShader.assets/image-20220514062029408.png" alt="image-20220514062029408" style="zoom:80%;" />
      1. ![image-20220514062109370](UnityShader.assets/image-20220514062109370.png)

#### 6. 矢量的叉积（cross product）外积（outerproduct）

1. 叉积结果是一个向量(vector)公式如下![image-20220514072216216](UnityShader.assets/image-20220514072216216.png)
   1. 叉积不满足交换律$a \times b \neq b \times a$，
   2. 不满足结合律$a \times (b \times c) \neq (a \times b) \times c$
   3. 但是满足**反交换律** $a \times b = -(b \times a)$

​     















































