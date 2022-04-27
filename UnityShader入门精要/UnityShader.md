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

<img src="UnityShader.assets/image-20220427091543453.png" alt="image-20220427091543453" style="zoom:80%;" />

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





































































