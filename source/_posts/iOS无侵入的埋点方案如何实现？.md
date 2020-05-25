---
title: iOS无侵入的埋点方案如何实现？
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-24 22:49:57
password:
summary:
tags:
categories:
---


在开发过程中，埋点可以解决两大类问题：一是了解用户使用 App 的行为，二是降低分析线上问题的难度。目前，iOS 开发中常见的埋点方式，主要包括：

- 代码埋点
- 可视化埋点
- 无埋点

### 代码埋点
代码埋点主要就是通过手写代码的方式来埋点，能很精确的在需要埋点的代码处加上埋点的代码，可以很方便地记录当前环境的变量值，方便调试，并跟踪埋点内容，但存在开发工作量大，并且埋点代码到处都是，后期难以维护等问题。  

#### 缺点：
1. 显而易见，你会在后期维护的时候写的怀疑人生
2. 复用性差，几乎不能移植给其他项目
3. 工作量大，而且会越写越多
4. 统计代码上线之后，如果出现问题，只能后续版本迭代
5. 如果统计项目名字改变了，原来老的APP版本依旧会统计老的页面名字

#### 优点：
1. 如果非要写一个其他统计无法做到的优点的话，那就是可自定义程度高吧，统计代码想写到那里写到那里（其实这些也可以在后面的方案实现，只是实现上稍微麻烦一点罢了）
2. 最容易想到的方案（前期费时少，使用起来费手不费思路）

### 可视化埋点
就是将埋点增加和修改的工作可视化了，提升了增加和维护埋点的体验。
#### 该方案的具体步骤就是：
1. 从后台获取需要统计的地方
2. hook住需要统计的类的load方法来Method Swizzing要统计的方法
3. 上传统计到的事件给后台分析


用```UIViewController```、```UIControl```为例子，讲解一下该方案的思路。

UIViewController PV统计,页面的统计较为简单，利用Method Swizzing hook 系统的viewDidLoad， 直接通过页面名称即可锁定页面的展示代码如下：

``` Swift
+(void)load { 
     static dispatch_once_t onceToken; 
     dispatch_once(&onceToken, ^{ 
     SEL originalDidLoadSelector = @selector(viewDidLoad); 
     SEL swizzingDidLoadSelector = @selector(analytic_viewDidLoad); 
     [MethodSwizzingTool swizzingForClass:[self class] originalSel:originalDidLoadSelector swizzingSel:swizzingDidLoadSelector]; 
}); 
}
 -(void)analytic_viewDidLoad {
    [self  analytic_viewDidLoad]; 
    //用当前类的类名作为统计页面的标识符  
    NSString * identifier = [NSString stringWithFormat:@"%@", [self class]];
     //通过当前类名获取PAGEPV表内的对应的页面的pageid和pagename  
     NSDictionary * dic = [[[AnalyticTool shareInstance].data objectForKey:@"PAGE"] objectForKey:identifier]; 
     if (dic) { 
     NSString * pageid = dic[@"screenData"][@"pageid"]; 
     NSString * pagename = dic[@"screenData"][@"pagename"]; 
     [AnalyticTool upLoadScreenName:pagename withScreenID:pageid]; 
     }
}
``` 

UIControl 点击统计,主要通过hook sendAction:to:forEvent: 来实现, 其唯一标识符我们用 targetname/selector/tag来标记，具体代码如下：

``` Swift
+(void)load 
{ 
    static dispatch_once_t onceToken; 
    dispatch_once(&onceToken, ^{ 
    SEL originalSelector = @selector(sendAction:to:forEvent:); 
    SEL swizzingSelector = @selector(analytic_sendAction:to:forEvent:); 
    [MethodSwizzingTool swizzingForClass:[self class] originalSel:originalSelector swizzingSel:swizzingSelector]; 
    }); 
}

-(void)analytic_sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event 
{ 
     [self analytic_sendAction:action to:target forEvent:event];
     NSString * identifier = [NSString stringWithFormat:@"%@/%@/%ld", [target class], 
     NSStringFromSelector(action),self.tag]; NSDictionary * dic = [[[AnalyticTool shareInstance].data objectForKey:@"ACTION"] objectForKey:identifier]; 
     if (dic) {
     NSString * eventid = dic[@"ActionData"][@"eventid"]; 
     NSString * targetname = dic[@"ActionData"][@"target"]; 
     NSString * pageid = dic[@"ActionData"][@"pageid"]; 
     NSString * pagename = dic[@"ActionData"][@"pagename"];
     [AnalyticTool upLoadActionEventWithScreenName:pagename withScreenID:pageid withTargetName:targetname withEventID:eventid]; 
    }
}
```
#### 缺点：
1. 需要后台配合
2. 可拓展性不是很高，因为需要修改后台下发的统计内容来每次的版本统计扩展

#### 优点：
1. 相对于第一种方案，代码量少了很多。
2. 动态化从后台获取统计内容，方便线上修改

### 无埋点
无埋点，并不是不需要埋点，而更确切地说是“全埋点”，而且埋点代码不会出现在业务代码中，容易管理和维护。它的缺点在于，埋点成本高，后期的解析也比较复杂，再加上 view_path 的不确定性。所以，这种方案并不能解决所有的埋点需求，但对于大量通用的埋点需求来说，能够节省大量的开发和维护成本。

在这其中，可视化埋点和无埋点，都属于是无侵入的埋点方案，因为它们都不需要在工程代码中写入埋点代码。所以，采用这样的无侵入埋点方案，既可以做到埋点被统一维护，又可以实现和工程代码的解耦。

接下来，我们就通过今天这篇文章，一起来分析一下无侵入埋点方案的实现问题吧。

### 运行时方法替换方式进行埋点
我们都知道，在 iOS 开发中最常见的三种埋点，就是对页面进入次数、页面停留时间、点击事件的埋点。对于这三种常见情况，我们都可以通过运行时方法替换技术来插入埋点代码，以实现无侵入的埋点方法。具体的实现方法是：先写一个运行时方法替换的类 ```ViewHook```，加上替换的方法 ```hookClass:fromSelector:toSelector```，代码如下：

``` Swift

#import "ViewHook.h"
#import <objc/runtime.h>

@implementation ViewHook

+ (void)hookClass:(Class)classObject fromSelector:(SEL)fromSelector toSelector:(SEL)toSelector {
    Class class = classObject;
    // 得到被替换类的实例方法
    Method fromMethod = class_getInstanceMethod(class, fromSelector);
    // 得到替换类的实例方法
    Method toMethod = class_getInstanceMethod(class, toSelector);
    
    // class_addMethod 返回成功表示被替换的方法没实现，然后会通过 class_addMethod 方法先实现；返回失败则表示被替换方法已存在，可以直接进行 IMP 指针交换 
    if(class_addMethod(class, fromSelector, method_getImplementation(toMethod), method_getTypeEncoding(toMethod))) {
      // 进行方法的替换
        class_replaceMethod(class, toSelector, method_getImplementation(fromMethod), method_getTypeEncoding(fromMethod));
    } else {
      // 交换 IMP 指针
        method_exchangeImplementations(fromMethod, toMethod);
    }

}

@end
  
``` 
这个方法利用运行时```  method_exchangeImplementations```  接口将方法的实现进行了交换，原方法调用时就会被```  hook```  住，从而去执行指定的方法。

**页面进入次数、页面停留时间都需要对 UIViewController 生命周期进行埋点**，你可以创建一个 UIViewController 的 Category，代码如下：

``` Swift

@implementation UIViewController (logger)
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 通过 @selector 获得被替换和替换方法的 SEL，作为 ViewHook:hookClass:fromeSelector:toSelector 的参数传入 
        SEL fromSelectorAppear = @selector(viewWillAppear:);
        SEL toSelectorAppear = @selector(hook_viewWillAppear:);
        [ViewHook hookClass:self fromSelector:fromSelectorAppear toSelector:toSelectorAppear];
        
        SEL fromSelectorDisappear = @selector(viewWillDisappear:);
        SEL toSelectorDisappear = @selector(hook_viewWillDisappear:);
        
        [ViewHook hookClass:self fromSelector:fromSelectorDisappear toSelector:toSelectorDisappear];
    });
}

- (void)hook_viewWillAppear:(BOOL)animated {
    // 先执行插入代码，再执行原 viewWillAppear 方法
    [self insertToViewWillAppear];
    [self hook_viewWillAppear:animated];
}
- (void)hook_viewWillDisappear:(BOOL)animated {
    // 执行插入代码，再执行原 viewWillDisappear 方法
    [self insertToViewWillDisappear];
    [self hook_viewWillDisappear:animated];
}

- (void)insertToViewWillAppear {
    // 在 ViewWillAppear 时进行日志的埋点
    [[[[SMLogger create]
       message:[NSString stringWithFormat:@"%@ Appear",NSStringFromClass([self class])]]
      classify:ProjectClassifyOperation]
     save];
}
- (void)insertToViewWillDisappear {
    // 在 ViewWillDisappear 时进行日志的埋点
    [[[[SMLogger create]
       message:[NSString stringWithFormat:@"%@ Disappear",NSStringFromClass([self class])]]
      classify:ProjectClassifyOperation]
     save];
}
@end
``` 
可以看到，``` Category```  在```  +load()```  方法里使用了 ViewHook 进行方法替换，在替换的方法里执行需要埋点的方法 ``` [self insertToViewWillAppear]。``` 这样的话，每个``` UIViewController```  生命周期到了```  ViewWillAppear ``` 时都会去执行```  insertToViewWillAppear```  方法。

那么，我们要怎么区别不同的 ``` UIViewController ``` 呢？我一般采取的做法都是，使用```  NSStringFromClass([self class])```  方法来取类名。这样，我就能够通过类名来区别不同的 ``` UIViewController```  了。

**对于点击事件来说，我们也可以通过运行时方法替换的方式进行无侵入埋点**。这里最主要的工作是，找到这个点击事件的方法 sendAction:to:forEvent:，然后在 +load() 方法使用 ViewHook 替换成为你定义的方法。完整代码实现如下：

``` Swift

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 通过 @selector 获得被替换和替换方法的 SEL，作为 ViewHook:hookClass:fromeSelector:toSelector 的参数传入
        SEL fromSelector = @selector(sendAction:to:forEvent:);
        SEL toSelector = @selector(hook_sendAction:to:forEvent:);
        [ViewHook hookClass:self fromSelector:fromSelector toSelector:toSelector];
    });
}

- (void)hook_sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
    [self insertToSendAction:action to:target forEvent:event];
    [self hook_sendAction:action to:target forEvent:event];
}
- (void)insertToSendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
    // 日志记录
    if ([[[event allTouches] anyObject] phase] == UITouchPhaseEnded) {
        NSString *actionString = NSStringFromSelector(action);
        NSString *targetName = NSStringFromClass([target class]);
        [[[SMLogger create] message:[NSString stringWithFormat:@"%@ %@",targetName,actionString]] save];
    }
}
``` 

和 UIViewController 生命周期埋点不同的是，UIButton 在一个视图类中可能有多个不同的继承类，相同 UIButton 的子类在不同视图类的埋点也要区别开。所以，我们需要通过 “action 选择器名NSStringFromSelector(action)” +“视图类名 NSStringFromClass([target class])”组合成一个唯一的标识，来进行埋点记录。

除了 UIViewController、UIButton 控件以外，Cocoa 框架的其他控件都可以使用这种方法来进行无侵入埋点。以 Cocoa 框架中最复杂的 UITableView 控件为例，你可以使用 hook setDelegate 方法来实现无侵入埋点。另外，对于 Cocoa 框架中的手势事件（Gesture Event），我们也可以通过 hook initWithTarget:action: 方法来实现无侵入埋点。

## 事件唯一标识
通过运行时方法替换的方式，我们能够 hook 住所有的 Objective-C 方法，可以说是大而全了，能够帮助我们解决绝大部分的埋点问题。

但是，这种方案的精确度还不够高，还无法区分相同类在不同视图树节点的情况。比如，一个视图下相同 UIButton 的不同实例，仅仅通过 “action 选择器名”+“视图类名”的组合还不能够区分开。这时，我们就需要有一个唯一标识来区分不同的事件。接下来，我就跟你说说**如何制定出这个唯一标识。**

这时，我首先想到的就是，能不能通过视图层级的路径来解决这个问题。因为每个页面都有一个视图树结构，通过视图的 superview 和 subviews 的属性，我们就能够还原出每个页面的视图树。视图树的顶层是 UIWindow，每个视图都在树的子节点上。如下图所示：

![cbfb127db8ed2545fd3ce0aa3ae6f452.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3eba9c53c6c8.png)

一个视图下的子节点可能是同一个视图的不同实例，比如上图中 UIView 视图节点下的两个 UIButton 是同一个类的不同实例，所以光靠视图树的路径还是没法唯一确定出视图的标识。那么，这种情况下，我们又应该如何区别不同的视图呢？

这时，我们想到了索引：每个子视图在父视图中都会有自己的索引，所以如果我们再加上这个索引的话，每个视图的标识就是唯一的了

接下来的一个问题是，视图层级路径加上在父视图中的索引来进行唯一标识，是不是就能够涵盖所有情况了呢？

当然不是。我们还需要考虑类似 UITableViewCell 这种具有可复用机制的视图，Cell 会在页面滚动时不断复用，所以加索引的方式还是没法用。

但这个问题也并不是无解的。UITableViewCell 需要使用 indexPath，这个值里包含了 section 和 row 的值。所以，我们可以通过 indexPath 来确定每个 Cell 的唯一性。

除了 UITableViewCell 这种情况之外， UIAlertController 也比较特殊。它的特殊性在于视图层级的不固定，因为它可能出现在任何页面中。但是，我们都知道它的功能区分往往通过弹窗内容来决定，所以可以通过内容来确定它的唯一标识。

除此之外，还有更多需要特殊处理的情况，但我们总是可以通过一些办法去确定它们的唯一性，所以我在这里也就不再一一列举了。思路上来说就是，想办法找出元素间不相同的因素然后进行组合，最后形成一个能够区别于其他元素的标识来。

除了上面提到的这些特殊情况外，还有一种情况使得我们也难以得到准确的唯一标识。如果视图层级在运行时会被更改，比如执行 insertSubView:atIndex:、removeFromSuperView 等方法时，我们也无法得到唯一标识，即使只截取部分路径也无法保证后期代码更新时不会动到这个部分。就算是运行时视图层级不会修改，以后需求迭代页面更新频繁的话，视图唯一标识也需要同步的更新维护。

这种问题就不好解决了，事件唯一标识的准确性难以保障，这也是通过运行时方法替换进行无侵入埋点很难在各个公司全面铺开的原因。虽然无侵入埋点无法覆盖到所有情况，全面铺开面临挑战，但是无侵入埋点还是解决了大部分的埋点需求，也节省了大量的人力成本。

最好的方案永远是针对于不同的场景来说的，我们不可能在一个创业团队一开始就选择方案3的架构，所以对于你来说，你要自己抉择目前而言对你最好的方案，如果你没有后台业务同学的支持，方案1也许对你来说真的是最好的方案了，起码是可以完成统计需求，虽然苦点累点。但是在合适的时间，切换不同的选择，才是成长的体现，还是最开始的话，如果你在的团队，已经给你了资源和时间去完善埋点这个模块，如果你把它做的更好，那一定是一件很酷的事情。

## 参考资料
 
1. [网易HubbleData无痕埋点SDK实现](https://neyoufan.github.io/2017/04/19/ios/%E7%BD%91%E6%98%93HubbleData%E6%97%A0%E5%9F%8B%E7%82%B9SDK%E5%9C%A8iOS%E7%AB%AF%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/)
2. [iOS无埋点数据SDK实践之路](https://www.jianshu.com/p/69ce01e15042)
3. [美团前端无痕埋点方案](https://tech.meituan.com/2017/03/02/mt-mobile-analytics-practice.html)
4. [微信读书团队Aspects的基本原理](https://wereadteam.github.io/2016/06/30/Aspects/)
5. [iOS打点杂谈](http://ayeio.com/ios/2017/03/19/%E5%85%B3%E4%BA%8E-iOS%E6%89%93%E7%82%B9%E6%9D%82%E8%B0%88.html)
