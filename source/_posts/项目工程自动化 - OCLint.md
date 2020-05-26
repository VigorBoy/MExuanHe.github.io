---
title: OCLint
date: 2018-1-26 00:48:32
tags:
---
## 说明
为了保证代码质量，Code Review 是非常重要的一环，受限于现实情况，大多数团队没有足够的时间进行 Code Review，那么只能把一部分 CR 工作交给计算机去完成了。我们只需要定下合理的流程，用代码告诉计算机需要做什么，剩下的就交给我们可靠的伙伴吧。

应用了自动化 Code Review 后，如果你的代码写得不好，Xcode 会表示不开心。
![16a34c6e6df309f1.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34c6e6df309f1.png)

如果你忽略 Xcode 的心情，那么质量管理平台会默默地记录这一切

![1724cbe9d51d2a49.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/1724cbe9d51d2a49.png)

这套东西既帮助开发们写出更高质量的的代码，也给经理们对工程质量的评估提供了一个切面的支持，同时只需要花费较少的人力维护

## OCLint
**工欲善其事，必先利其器**

[OCLint](http://oclint.org)是一个开源的，基于 Clang 用 C++ 编写而成的，可以用于 C、C++ 和 Objective-C 的静态代码分析器。它可以在扫描的过程中动态加载规则文件（Rules），因此可以实现非常灵活的，高度可自定义的代码分析方案。它几乎可以和大多数系统无缝集成，例如 Cmake、Bear、xcodebuild、xctool、Xcode、xcpretty、Jenkins CI、Travis CI 等。

OCLint 通过代码并寻找潜在问题来提高质量和减少缺陷，比如：

1. 可能的错误 - 清空if / else / try / catch / finally语句
2. 未使用的代码 - 未使用的本地变量和参数
3. 复杂的代码 - 高回圈复杂度，NPath复杂度和高NCSS
4. 冗余代码 - 冗余如果语句和无用的括号
5. 代码气味 - 长方法和长参数列表
6. 不好的做法 - 倒逻辑和参数重新分配
...

## OCLint 安装
确保你已经安装了 [Homebrew](https://brew.sh/index_zh-cn)

```
$ brew tap oclint/formulae

$ brew install oclint
```

## 更新 OCLint

```
$ brew update

$ brew upgrade oclint
```

安装好后在终端中输入 oclint 验证是否成功安装，如出现如下提示说明已安装成功：

```
$ oclint

oclint: Not enough positional command line arguments specified!

Must specify at least 1 positional arguments: See: oclint -help
```

## 使用本地 Review

1. 首先在电脑本地安装好 OCLint 并拿到公司自定义的 Rules 文件
2. [在 Xcode 上配置好工程](http://docs.oclint.org/en/stable/guide/xcode.html)

![16a34c6e7375e297.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34c6e7375e297.png)
![16a34c6e73e62ce7.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34c6e73e62ce7.png)
![16a34c6e72b9d018.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34c6e72b9d018.png)
![16a34c6e73d928ad.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34c6e73d928ad.png)

3. build工程，等待结果显示在Xcode上。