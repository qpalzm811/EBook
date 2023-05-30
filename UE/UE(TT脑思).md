# 1.入门

## 1.设置人物出生点，两种方式

1. 用Player Start出生点设置![image-20221127205809152](UE(TT脑思).assets/image-20221127205809152.png)
2. 将模型放在场景里，选择人物作为出生操作对象
   1. 删除Player Start 点
   2. 选择人物模型——Detail——搜索auto(Pawn页签下 Auto Possess Player)——选择Player 0
   3. ![image-20221127211945700](UE(TT脑思).assets/image-20221127211945700.png)

## 2.切换GameMode

WorldSetting窗口——GameMode——GameMode Override选择视角蓝图，Select GameMode 默认pawn类选择角色（pawn就是主角长啥样子

![image-20221128024458503](UE(TT脑思).assets/image-20221128024458503.png)

## 3.生成静态网格体

![image-20221205084854837](UE(TT脑思).assets/image-20221205084854837.png)

### 3.1增加静态网格体碰撞

双击静态网格体，打开如下界面

![image-20221205091253879](UE(TT脑思).assets/image-20221205091253879.png)

## 4.新建地形 植物（进去就知道了）

![image-20221214031409547](UE(TT脑思).assets\image-20221214031409547.png)

## 5.雾 光

### 5.1 能实现体积光效果

指数级高度雾——体积雾——光束遮挡

![image-20221214032057040](UE(TT脑思).assets\image-20221214032057040.png)

### 5.3 后处理体积

![image-20221214033744779](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214033744779.png)

#### 光强 bloom exposure

![image-20221214034238375](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214034238375.png)

#### Chromatic Aberration

![image-20221214034433385](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214034433385.png)

#### dirt mask 

![image-20221214035305967](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214035305967.png)

#### Lens flares

镜头前的点

![image-20221214040204027](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214040204027.png)

![image-20221214041813925](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214041813925.png)

## 6.粒子系统

![image-20221214042504934](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214042504934.png)

设置粒子范围

![image-20221214042524869](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214042524869.png)

设置粒子发光，把rgb调大过1，越大，粒子越大

![image-20221214042830727](UE(TT%E8%84%91%E6%80%9D).assets/image-20221214042830727.png)

## 7. 添加声音

拖到中间，点自动播放就能一直有

还能选择渐变衰退声音

## 8.动画



![image-20221225135533446](UE(TT%E8%84%91%E6%80%9D).assets/image-20221225135533446.png)
