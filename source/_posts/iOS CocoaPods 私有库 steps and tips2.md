---
title: iOS CocoaPods 私有库 steps and tips
date: 2017-01-18 17:28:33
tags:
categories: ios
---
# iOS CocoaPods 私有库 steps and tips
这篇文章记录了下cocoapods私有库的一些steps和tips.

* 如何创建私有仓库
* 如何创建私有库

<!-- more -->

# iOS CocoaPods 私有库 steps and tips
这篇文章记录了下cocoapods私有库的一些steps和tips.

* 如何创建私有仓库
* 如何创建私有库
* 如何更新发布私有库
* 如何使用私有库


# 如何创建私有仓库

## steps
先随便在网上创建一个仓库, 然后在本地添加那个那个仓库.

```
pod repo add PHSpecs http://pagit.paic.com.cn/PHiOSFootstone/PHSpecs.git
```

# 如何创建私有库

## steps
* 创建私有库模板

```
pod lib create VenderName
```
创建模板的时候会问你要用什么语言, 要不要自带测试框架什么的.

生成的模板结构如下:

```
$ tree VenderName -L 2
VenderName
├── Example                                   #demo APP
│   ├── VenderName
│   ├── VenderName.xcodeproj
│   ├── VenderName.xcworkspace
│   ├── Podfile                               #demo APP 的依赖描述文件
│   ├── Podfile.lock
│   ├── Pods                                   #demo APP 的依赖文件
│   └── Tests
├── LICENSE                                #开源协议 默认MIT
├── Pod                                        #组件的目录
│   ├── Assets                             #资源文件
│   └── Classes                               #类文件
├── VenderName.podspec            #第三步要创建的podspec文件
└── README.md                                 #markdown格式的README
9 directories, 5 files
```

## tips

* 引用静态库：'(.ios).library'。去掉头尾的lib，用","分割

```
// 引用libxml2.lib和libz.lib.
spec.libraries =  'xml2', 'z'
```

* 引用公有framework：'(.ios).framework'. 用","分割. 去掉尾部的".framework"

```
spec.frameworks = 'UIKit','SystemConfiguration', 'Accelerate' 
```

* 引用自己生成的framework：'(.ios).vendored_frameworks'。用","分割
路径写从.podspec所在目录为根目录的相对路径 ps:这个不要省略.framework

```
spec.ios.vendored_frameworks = 'Pod/Assets/*.framework'
```

* 引用自己生成的.a文件, 添加到Pod/Assets文件夹里. Demo的Example文件夹里也需要添加一下, 不然找不到
```
spec.ios.vendored_libraries = 'Pod/Assets/*.a'
```
在提交到私有仓库的时候需要加上`--use-libraries`

# 如何更新发布私有库

## steps
* 提交代码

```
git add -A && git commit -m "Release 1.0.0"
```

* 打tag

```
git tag '1.0.0'
```

* 把tag推到远程仓库

```
git push --tags
```

* 将本地的master分支推送到远程仓库

```
git push origin master
```

* 提交到私有仓库

```
pod repo push PHSpecs VenderName.podspec --allow-warnings --verbose
// --allow-warnings : 允许 警告，有一些警告是代码自身带的。
// --use-libraries  : 私有库、静态库引用的时候加上
// —verbose ： lint显示详情
```

## tips

提交到私有仓库的之前可以先验证一下, 有问题就修复它, 验证过了在提交

```
pod spec lint VenderName.podspec --verbose
```

打好tag, 推到git里去后, `才可以`在测试的项目里的Podfile里引用这个库, 然后 `pod update VenderName --no-repo-update`, 测试通过了, 在提交到私有仓库里

```
pod 'VenderName', :podspec => '/Users/heaven/Desktop/work/VenderName/VenderName.podspec'
```

还可以指定引用某个分支的代码

```
pod 'VenderName', :git => 'https://github.com/PHiOSFootstone/VenderName.git', :branch => 'develop'
```

提交到私有仓库的时候还可以忽略警告类的错误, 愣是要提交. 在后面加上 `--allow-warnings`

```
pod repo push PHSpecs VenderName.podspec --allow-warnings
```

如果有添加新的文件, 需要更新下引索, demo里才可以识别

```
pod update VenderName --no-repo-update
```

* 资源用bundle, 添加到Pod/Assets文件夹里. Demo的Example文件夹里也需要添加一下, 不然找不到

```
// podspec里
s.resources = "Pod/Assets/*"

// 代码里
[UIImage imageNamed:@"VenderName.bundle/pic1.png"]
```

# 如何使用私有库

## steps

* 添加私有仓库

```
pod repo add PHSpecs http://pagit.paic.com.cn/PHiOSFootstone/PHSpecs.git
```

* 用的时候需要在Podfile里添加源

```
// github地址
source 'https://github.com/CocoaPods/Specs.git'
# ...相关库

// 私有库地址
source 'http://pagit.paic.com.cn/PHiOSFootstone/PHSpecs.git'
# ...私有库
```
* 用的时候在Podfile里引用

```
pod 'VenderName'
```

## tips
使用的时候还可以通过直接指定地址 + tag or 分支 or commit 的方式来引入, 这样就可以不用走发布流程了. 也不需要添加源了.

```
pod 'VenderName', :git => 'http://pagit.paic.com.cn/PHiOSFootstone/VenderName.git', :tag => '0.7.0'

pod 'VenderName', :git => 'http://pagit.paic.com.cn/PHiOSFootstone/VenderName.git', :branch => 'develop'

pod 'VenderName', :git => 'http://pagit.paic.com.cn/PHiOSFootstone/VenderName.git', :commit => '082f8319af'
```

查看私有仓库

```
cd ~/.cocoapods/repos/
```
