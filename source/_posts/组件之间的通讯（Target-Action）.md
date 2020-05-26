---
title: 组件之间的通讯（Target-Action）
date: 2019-2-16 00:46:02
tags:
---
# 什么是组件间通讯?
- 比如现在有很多业务组件, 在另外一个组件内部需要调用另外一个组件中的服务, 或者打开另外一个组件中的控制器, 并传值

**iOS业界讨论组件化方案甚多，大体来说有3种。**
- Protocol注册方案
- URL注册方案
- Target-Action runtime调用方案


#### MGJRoute方案

URL注册方案 **[蘑菇街 App 的组件化之路](https://link.jianshu.com/?t=http://limboy.me/tech/2016/03/10/mgj-components.html)** 已经说的很清楚了 可以去看下

原理：
- 通过url注册服务, 其他地方通过url, 获取服务
- 框架在维护一个url-block的表格

特点：
- 每个业务组件, 都需要依赖这个框架
- url维护成本高 硬解码
- 可以在组件内部任何地方调用/注册服务, 没有必要统一组件接口服务


#### target-action方案
原理：
- 每个组件, 提供一个统一披露的接口文件
- 额外的维护一个中间件的分类扩展（在此处进行硬解码 通过运行时进行物理解耦）
- 其他地方通过target-action;的方案进行交互

特点：
- 集约
- 统一了组件api服务
- 组件与框架之间无依赖关系
- 需要额外维护中间件类扩展

#### Protocol方案  暂无了解

## 本文 主要讲解 [target-action](https://github.com/MExuanHe/Target-Action-Dome) 方案 ## 


**侵入性问题**
正如你所见，CTMediator组件化方案的实施非常安全。因为它并不存在任何侵入性的代码修改。
对于响应者来说，什么代码都不用改，只需要包一层Target-Action即可。
对于调用者来说，只需要把调用方式换成CTMediator调用即可，其改动也不涉及原有的业务逻辑，所以是十分安全的。

**注册问题**
CTMediator没有任何注册逻辑的代码，避免了注册文件的维护和管理。Category给到的方法很明确地告知了调用者应该如何调用。

例如给到的
 ```
- (UIViewController *)goodsDetailViewControllerWithGoodsId:(NSString *)goodsId goodsName:(NSString *)goodsName;
``` 
方法。这能够让工程师一眼就能够明白使用方式，而不必抓瞎拿着URL再去翻文档。
这可以很大程度提高工作效率，同时降低维护成本。

下面是我做的项目Dome结构![Snip20180403_5.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cb1e0b80dee.png)

我们主要是依赖``` CTMediator```  这个中间件 

工具类中主要使用如下方法
``` 
- (id)performTarget:(NSString *)targetName action:(NSString *)actionName params:(NSDictionary *)params shouldCacheTarget:(BOOL)shouldCacheTarget
``` 
方法内部使用Runtime调用  需要传三个参数 
- 当前需要调用的类名  （字符串）
- 当前需要调用类的方法名 （字符串）
- 需要传的参数 （字典形式） 

``` 
# 通过Runtime  把字符串 转换类
Class targetClass = NSClassFromString(ClassString);
id  target = [[targetClass alloc] init];

# 把字符串转换成事件
SEL action = NSSelectorFromString(actionString);

# 如果当前类中有这个事件 那就执行这个事件 把需要的参数传值 
if ([target respondsToSelector:action]) {
    return [target performSelector:action withObject:params];
} 
``` 

下面是一个组件的结构
![Snip20180403_6.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cb1e5814e97.png)

我们当前 ``` TAConfirmOrder```  组件中
- ```TAConfirmOrderViewController``` 是业务组件 
- ```Target_TAConfirmOrder``` 是每个组件, 提供一个统一披露的接口文件  
- ```CTMediator+TAConfirmOrder``` 是额外的维护一个中间件的分类扩展
- 组件与框架之间无依赖关系，我们需要额外维护中间件类扩展就可以了


我们只需要把类名和类中的方法名 告诉这个分类扩展就行了
![Snip20180403_7.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cb1e570d952.png)


最后  [Dome 地址](https://github.com/MExuanHe/Target-Action-Dome) 