---
title: 使用SceneKit框架来加载3D引擎
date: 2019-12-26 00:16:57
tags:
---
## 简介：
使用高级场景描述创建3D游戏并将3D内容添加到应用程序。轻松添加动画，物理模拟，粒子效果和基于物理的逼真的渲染。
[SceneKit](https://developer.apple.com/documentation/scenekit)将高性能渲染引擎与描述性API结合在一起，用于导入，操作和渲染3D资源。与要求您精确实现显示场景的渲染算法的低级API（例如Metal和OpenGL）不同，SceneKit只需要描述场景的内容以及想要执行的动作或动画。

在使用SceneKit之前，应该熟悉基本的图形概念，例如坐标系和三维几何的数学。SceneKit使用右手坐标系，其中（默认情况下）视图方向沿负z轴，如下所示。
![03f7f401-5c21-4ac9-9c8c-e7640ec11d81.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e26c69eafb3.png)
##### 开始本小节之前，科普一下两种文件格式：
- .scn 文件，是SceneKit的 原生格式，这种格式的场景文件支持所有的SceneKit特性，诸如物理效果、约束以及粒子系统，并且，按照这种格式来导入场景文件，速度是最快的
- .dae 文件，dae是Digital Asset Exchange 的缩写，这种格式的场景文件不支持SceneKit特性，诸如物理效果、约束以及粒子系统

### 在开发当中常用的SceneKit类
![WeChat1dbfaae3566f808731bda7d577c1de88.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e26cd2d131d.png)

## SCNScene:
要使用SceneKit显示3D内容，需要创建一个场景图，其中包含节点和属性的层次结构，这些层次结构一起代表了可视元素。
##### 示例
``` Swift
let url:URL = Bundle.main.url(forResource: "ship", withExtension: ".dae")!
let scene:SCNScene = try! SCNScene(url: url, options: nil)
```

## SCNView：

在macOS中，SCNView是的子类[NSView](https://developer.apple.com/documentation/appkit/nsview)。在iOS和tvOS中，SCNView是的子类[UIView](https://developer.apple.com/documentation/uikit/uiview)。作为这两种操作系统的视图层次结构的一部分，一个SCNView对象为应用程序的用户界面中的SceneKit内容提供了一个位置。
##### 示例
``` Swift
let scnView:SCNView = SCNView(frame: CGRect.zero)
scnView.backgroundColor = UIColor.clear
scnView.scene = scene
self.view.addSubview(scnView)
``` 

##### 常用功能：
- 设置帧率
- 截屏
- 开始和暂停游戏
- 抗锯齿
- 控制摄像机
- 显示统计菜单
- 执行渲染方式(OpenGL /Metal)

## SCNNode:

场景图的结构元素，表示3D坐标空间中的位置和变换，可以在其中附加几何图形，灯光，照相机或其他可显示内容。节点在场景中都以树状结构存在。

说白了，节点就是一个抽象的东东，看不见也摸不着，但是有着自己的坐标系和位置。它与场景（SCNView）、游戏元素（比如灯光，摄像机和几何图形）的关系是：游戏元素要呈现出来，不能直接添加到场景中，而是绑定到节点（SCNNode）上，再把节点添加到场景中，就能看到游戏元素了。

类似UIView 的 addSubView 的方法，SCNNode 可以通过addChildNode 方法去添加子节点。
Scene 中有一个特殊的节点：root node, 场景中所有的节点要么是根节点的子节点，要么就是根节点的子节点的子节点 

## SCNLight：
光源可以附加到节点上，在渲染场景中提供着色，负责整个场景的明暗，阴影

灯光的分类(光源分为四种):
- 环境光(SCNLightTypeAmbient),这种光没有方向,位置在无穷远处,光均匀的散射在物体上
- 点光源(SCNLightTypeOmni):有固定位置,方向360度,可以衰减
- 平行方向光(SCNLightTypeDirectional):只有照射的方向,没有位置,不会衰减
- 聚焦光源(SNCLightTypeSpot):光有固定位置,也有方向,也有照射区域,可以衰减
![1892971-bdcff22a8856ab60.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e26c6f05a87.png)

##### 示例：  
``` Swift
let ambientLightNode = SCNNode()
ambientLightNode.light = SCNLight()
ambientLightNode.light?.type = .ambient
ambientLightNode.light?.color = UIColor.white
```

## SCNCamera：

这个类似我们现实中的相机，它也有焦距、视角等。图形渲染到模型后，要添加相机我们才能看见。
![1f516915-005c-4949-9bc9-38a3fe9f2a7d.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e26cca315d9.png)

##### 示例：
``` Swift
let camera = SCNCamera()
camera.automaticallyAdjustsZRange = true
camera.zNear = 45
camera.zFar = 100
        
let cameranode = SCNNode()
cameranode.camera = camera
``` 
##### 常用属性：
- fieldOfView 视角，默认60°【值越小，看到的物体细节越在前面，即被放大】
- focalLength  焦距，默认50mm【值越小，看到的物体越远】
- zNear 相机能照到的最近距离，默认1m
- zFar 相机能照到的最远的距离，默认100m
- focusDistance 焦距 默认2.5 焦距越大，视角越小
- focalBlurSampleCount 设置聚焦时，模糊物体模糊度 默认0

## SCNGeometry：
SCeneKit 游戏框架中的几何对象.将几何对象绑定到节点上,显示到view

##### 系统包含的：
- 正方体,
- 平面(SCNPlane),
- 金字塔,
- 球体,
- 圆柱体,
- 圆锥体,
- 管道,
- 环面,
- 地板(SCNFloor),
- 立体字,
- 自定义形状(通过贝塞尔曲线)创建SCNShape
然后赋值给Node 节点
![图片 1.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e26cca315d9.png)

##### 示例：
``` Swift
// 1. 球体
SCNSphere *sphere = [SCNSphere sphereWithRadius:0.5];
sphere.firstMaterial.diffuse.contents = @"earth.jpg";
SCNNode *earthNode = [SCNNode nodeWithGeometry:sphere];
//2. 字体
SCNText *scntext = [SCNText textWithString:@"xco" extrusionDepth:0.3];
scntext.font = [UIFont systemFontOfSize:0.3];
SCNNode *textnode = [SCNNode nodeWithGeometry:scntext];
textnode.position = SCNVector3Make(-1, 0, -2);
``` 

####  讲一下在SceneKit中的骨骼动画
苹果官方给的定义

>骨骼动画是一种简化复杂几何形状的动画的技术,比如游戏中人的特征,动画骨架是一个简单的控制节点的层次结构,本身没有可见的几何对象,将骨头和几何对象进行结合,当你移动这个骨头控制的节点时允许SceneKit 去自动使几何对象变形。

***如图：***
![1594482-3123f3954171796e.png.jpg](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e26c8ed534e.jpeg)
怎么使用骨骼动画
1. 一般情况下,游戏设计师使用3D工具创建一个皮肤模型,包含了骨骼的动画,保存在一个场景文件中,你从场景文件中导入这个骨骼模型,然后让他们运动起来
2. 另外你也可以直接从场景文件中导入动画对象直接操作骨头节点
3. 你还可以单独创建一个自定义的几何和骨架数据的皮肤模型

分析一下在Xcode中加入带有骨骼动画的3D模型的结构
![1594482-5ef45a7d2edecf3c.png.jpg](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e26fee43cde.jpeg)
![1594482-e2133816b7a0c830.png.jpg](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e270306081d.jpeg)
![1594482-b65196d4d64c5fa7.png.jpg](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e27184683f9.jpeg)

##### 完整的骨骼动画效果
![1594482-71551d5854d32b5b.gif](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3e2706e26d8f.gif)
