---
title: iOS降低APP崩溃率
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-24 22:53:08
password:
summary:
tags:
categories:
---

作为一个资深的技术团队，app的性能是我们技术团队首要的任务，其中最主要的一项就是app的崩溃率。
目前虽然不能把系统所有的crash都处理掉，不过一些常见的高频次发生的crash，系统都会处理。目前主要可以处理掉的crash类型有一下几种：

- unrecognized selector crash
- KVO crash
- NSNotification crash
- NSTimer crash
- Container crash（数组越界，插nil等）
- NSString crash （字符串操作的crash）
- UI not on Main Thread Crash (非主线程刷UI(机制待改善))
##### 下面会一一讲解如何解决这些carsh

## unrecognized selector crash
unrecognized selector类型的crash是经常发生的carsh，我们要解决这个carsh就必须先了解它产生的具体原因和流程。

什么时候会报unrecognized selector的异常？

objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，如果，在最顶层的父类中依然找不到相应的方法时，程序在运行时会挂掉并抛出异常unrecognized selector sent to XXX

在找不到方法时，查找方法将会进入方法Forward流程,系统给了三次补救的机会，所以我们要解决这个问题，在这三次均可以解决这个问题
![图片](/MExuanHe.github.io/fancybox/images/rootsearch.png)
由上图可见，在一个函数找不到时，runtime提供了三种方式去补救：

1、调用resolveInstanceMethod给个机会让类添加这个实现这个函数
2、调用forwardingTargetForSelector让别的对象去执行这个函数
3、调用forwardInvocation（函数执行器)灵活的将目标函数以其他形式执行。

如果都不中，调用doesNotRecognizeSelector抛出异常。既然可以补救,我们可以用消息转发机制来做，我们选择了第二步forwardingTargetForSelector来做，原因如下：

1、resolveInstanceMethod 需要在类的本身上动态添加它本身不存在的方法，这些方法对于该类本身来说冗余的

2、forwardInvocation可以通过NSInvocation的形式将消息转发给多个对象，但是其开销较大，需要创建新的NSInvocation对象，并且forwardInvocation的函数经常被使用者调用，来做多层消息转发选择机制，不适合多次重写

3、forwardingTargetForSelector可以将消息转发给一个对象，开销较小，并且被重写的概率较低，适合重写
选择了forwardingTargetForSelector之后，可以将NSObject的该方法重写

做以下几步的处理：

1、动态创建一个桩类

2、动态为桩类添加对应的Selector，用一个通用的返回0的函数来实现该SEL的IMP

3、将消息直接转发到这个桩类对象上。
流程图如下：
![图片2](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/170d3ef056e30b35.png)
注意如果对象的类本事如果重写了forwardInvocation方法的话，就不应该对forwardingTargetForSelector进行重写了，否则会影响到该类型的对象原本的消息转发流程。

通过重写NSObject的forwardingTargetForSelector方法，我们就可以将无法识别的方法进行拦截并且将消息转发到安全的桩类对象中，从而可以使app继续正常运行。

## KVO crash
如果观察者和keypath的数量一多，很容易理不清楚被观察对象整个KVO关系，导致被观察者在dealloc的时候，还残存着一些关系没有被注销。 同时还会导致KVO注册观察者与移除观察者不匹配的情况发生。
那么如何来管理混乱的KVO关系呢。可以让被观察对象持有一个KVO的delegate，所有和KVO相关的操作均通过delegate来进行管理，delegate通过建立一张map来维护KVO整个关系
这样做的好处有两个：

1、如果出现KVO重复添加观察者或重复移除观察者（KVO注册观察者与移除观察者不匹配）的情况，delegate可以直接阻止这些非正常的操作。

2、被观察对象dealloc之前，可以通过delegate自动将与自己有关的KVO关系都注销掉，避免了KVO的被观察者dealloc时仍然注册着KVO导致的crash。
被swizzle的方法分别是：
```Switch
- (void)addObserver:(NSObject *)observer 
			forKeyPath:(NSString *)keyPath
			 	options:(NSKeyValueObservingOptions)options 
			 	context:(nullable void *)context;
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath;
- (void)observeValueForKeyPath:(nullable NSString *)keyPath ofObject:(nullable id)object change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change context:(nullable void *)context;

```

关于
```Switch
- (void)addObserver:(NSObject *)observer 
			forKeyPath:(NSString *)keyPath
			 	options:(NSKeyValueObservingOptions)options 
			 	context:(nullable void *)context;

```
**方法改造流程如下图：**
![图片3.png](/MExuanHe.github.io/fancybox/images/170d3ef057740bea.png)
通过上面的流程，将observerd对象的所有kvo相关的observer信息全部转移到KVOdelegate上，并且避免了相同kvoinfo被重复添加多次的可能性。
关于：
```Switch
- (void)removeObserver:(NSObject *)observer
            forKeyPath:(NSString *)keyPath
               context:(void *)context
```
方法改造流程如下图：
![图片 4.png](/MExuanHe.github.io/fancybox/images/170d3ef058373d96.png)
移除一个keypath的Observer时，当delegate的kvoInfoMap中找不到key为该keypath的时候，说明此时delegate并没有持有对应keypath的observer，即说明移除了一个不匹配的观察者，此时如果再继续操作会导致app崩溃，所以应该及时中断流程，然后统计异常信息。

当keypath对应的KVOInfo列表（infoArray）为空的时候，说明此时delegate已经不再持有任何和keypath相关的observer了。这时应该调用原有removeObserver的方法将delegate对应的观察者移除。
注意到在检查遍历infoArray的时侯，除了要删除对应的info信息，还多了一步检查info.observer == nil的过程，是因为如果observer为nil，那么此时如果keypath对应的值变化的话，也会因为找不到observer而崩溃，所以需要做这一步来阻止该种情况的发生。
###### 关于
```Switch
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSString *,id> *)change
                       context:(void *)context
```
delegate对于observeValueForKeyPath方法的修改最主要的地法规，在于将对应的响应方法转移给真正的KVO Observer，通过keyInfoMap找到keypath对应的KVOInfo里面预先存储好的observer，然后调用observer原本的响应方法

同时在遍历InfoArray的时候，发现info.observerw == nil的时候，需要及时将其清除掉，避免KVO的观察者observer被释放后value变化导致的crash

最后，针对 KVO的被观察者dealloc时仍然注册着KVO导致的crash 的情况

可以将NSObject的dealloc swizzle， 在object dealloc的时候自动将其对应的kvodelegate所有和kvo相关的数据清空，然后将kvodelegate也置空。避免出现KVO的被观察者dealloc时仍然注册着KVO而产生的crash
## NSNotification crash

当一个对象添加了notification之后，如果dealloc的时候，仍然持有notification，就会出现NSNotification类型的crash。

利用method swizzling hook NSObject的dealloc函数，再对象真正dealloc之前先调用一下[[NSNotificationCenter defaultCenter] removeObserver:self] 即可。

注意到并不是所有的对象都需要做以上的操作，如果一个对象从来没有被NSNotificationCenter 添加为observer的话，在其dealloc之前调用removeObserver完全是多此一举。 所以我们hook了NSNotificationCenter的addObserver:(id)observer selector:(SEL)aSelector name:(NSString *)aName object:(id)anObject 函数，在其添加observer的时候，对observer动态添加标记flag。这样在observer dealloc的时候，就可以通过flag标记来判断其是否有必要调用removeObserver函数了。

## NSTimer crash

NSTimer存在以下问题:
•	Target是强引用，内存泄漏
•	在宿主不存在的时候，清理NSTimer

解决方法: Hook NSTimer中
``` 
scheduledTimerWithTimeInterval:target:selector:userInfo:repeats方法
```

1、当repeats为NO时，走原始方法
2、当repeats为YES时，新建一个对象，声明一个target属性为weak类型，指向参数的target,当中间对象的target为空时，清理NSTimer

## Container crash（数组越界，插nil等）

Container 类型的crash 指的是容器类的crash

**常见的有:**
- NSArray
- NSMutableArray
- NSDictionary
- NSMutableDictionary
- NSCache

一些常见的越界，插入nil，等错误操作均会导致此类crash发生

Container crash 类型的防护方案也比较简单,针对于NSArray／NSMutableArray／NSDictionary／NSMutableDictionary／NSCache的一些常用的会导致崩溃的API进行method swizzling，然后在swizzle的新方法中加入一些条件限制和判断，从而让这些API变的安全，这里就不展开来具体描述了。

## NSString crash （字符串操作的crash）

NSString／NSMutableString 类型的crash的产生原因和防护方案与Container crash很相像，这里也不展开来描述了。

## UI not on Main Thread Crash (非主线程刷UI)
在非主线程刷UI将会导致app运行crash，有必要对其进行处理。 目前初步的处理方案是swizzle UIView类的以下三个方法：

- (void)setNeedsLayout;
- (void)setNeedsDisplay;
- (void)setNeedsDisplayInRect:(CGRect)rect;

在这三个方法调用的时候判断一下当前的线程，如果不是主线程的话，直接利用
dispatch_async(dispatch_get_main_queue(), ^{
//调用原本方法
});

来将对应的刷UI的操作转移到主线程上，同时统计错误信息。

