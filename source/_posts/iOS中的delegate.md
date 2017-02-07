---
title: iOS中的delegated的用法和规范
date: 2017-01-18 17:28:33
tags:
categories: ios
---

# iOS中的delegated的用法和规范
delegate是Objective-C编写的应用中各种对象之间互相调用的主要方式之一. 通常情况下, 对象可以接收的消息都通过在接口文件中声明的方法来表示.

<!-- more -->
# iOS中的delegated的用法和规范
delegate是Objective-C编写的应用中各种对象之间互相调用的主要方式之一. 通常情况下, 对象可以接收的消息都通过在接口文件中声明的方法来表示.

```
@protocol MyProtocol <NSObject>
- (void)func;
@end
```

## 什么是delegate
delegate是委托模式.委托模式是将一件属于委托者做的事情，交给另外一个被委托者来处理.
一个标准的委托由以下部分组成:

### 协议的声明
我们需要用协议来申明哪些方法是被委托出去了.

```
@protocol  MyUIViewDelegate <NSObject>
- (void)func;
@end
```

### 委托者申明一个属性
委托者里得有一个属性代表被委托者, 注意这个属性是弱引用.

```
@interface  MyUIView： UIView
@property(nonatomic, weak) id<MyUIViewDelegate> delegate;
```

### 被委托者声明实现了协议
被委托者需要声明自己实现了委托里的协议.

```
@interface MyUIViewController : UIViewController <MyUIViewDelegate>
@end
```

### 设置委托
在被委托者里设置自己是委托者的被委托者.嘛,这句话是有些绕.

```
// viewcontroller. m中
- (id)init
{
	MyUIView *myView = [[MyUIView alloc] init];  //对MyUIView进行初始化
	myView.delegate = self;   // 将MyUIViewController自己的实例作为委托对象
	self.view = myView;
}
```

### 委托事件
在委托者里调用委托的方法.

```
// MyUIView.m中
- (void)doSomething
{
	[self.delegate func];  
}
```

## delegate的用途
委托一般可以分成3种
### 传递事件
传递事件就是A发生了什么事情, 希望B知道下, 然后B在自己的类里面要做出某些反应.典型的如
`tableView:didSelectRowAtIndexPath:`, 就是UITableView点击了某个cell的时候, 希望其它类(通常是ViewController)响应这个点击, 在点击的时候跳转到其他viewController之类的.

### 确定事件可执行
确定事件可执行是当A需要执行某个事件的时候, A不确定到底可执行, 这个时候希望B能回应下. 如`tableView:shouldHighlightRowAtIndexPath:`是UITableView询问其它类要不要高亮显示某个cell, 当返回NO的时候, 就UITableView就不会执行cell的高亮方法.

### 传递值
传递值是当A需要某个数据的时候, 由B来提供. 例子还是UITableView里的,`tableView:cellForRowAtIndexPath:`是需要某个cell的时候由其他类提供这个cell.

## 委托命名
### 委托
通常的委托用delegate做后缀.如`<UIScrollViewDelegate>`

```
@protocol <#class#>Delegate
```

### 数据源
当你的委托的方法过多, 可以拆分数据部分和其他逻辑部分, 数据部分用dataSource做后缀. 如`<UITableViewDataSource, UITableViewDelegate> `

```
@protocol <#class#>DataSource
```

## 方法修饰
委托的方法不是百分百必须实现的.
### 必须实现的方法
用required修饰的方法是必须实现的.协议默认声明在其中的方法为必须实现的方法.

```
@protocol MyProtocol <NSObject>

@required
- (void)func;

@end

// 用的时候
- (void)doSomething
{
	[_delegate <#func2#>];
}
```

### 可以实现的方法
用optional修饰的方法可以不实现. 在用到的时候需要先判断方法是否存在

```
@protocol MyProtocol <NSObject>

@optional
- (void)func;

@end

// 用的时候
- (void)doSomething
{
	if (_delegate respondsToSelector:@selector(<#func2#>))
	{
		[_delegate <#func2#>];
	}
}
```



## 方法命名
当特定的事件发生时, 对象会触发它注册的委托方法.

委托的方法, 第一个参数是触发它的对象，第一个关键词是触发对象的类名, 错误的状态必须带有error信息, 其他的参数看实际情况. 根据委托方法触发的时机和目的, 使用should,will,did等关键词.更具事件的状态, 使用finish, fail, start等关键词.

```
- (BOOL)tableView:(NSTableView *)tableView shouldSelectRow:(int)row;
```

### 完成
finish表示一个事件已经完成, 通常情况下我们默认是成功.

```
- (void)<#class#>DidFinish<#event#>:(id)class
```
### 失败
fail表示一个事件已经失败了, 我们在这里需要返回错误的原因.

```
- (void)<#class#>:(id)class didFail<#event#>:(NSError *)error
```
### 开始
start标志一个事件的开始.

```
- (void)<#class#>DidStart<#event#>:(id)class
```

### 将要开始
should表示某事件将要开始.同意开始则返回YES, 否则返回NO

```
- (BOOL)<#class#>ShouldStart<#event#>:(id)class
```

## 多播委托
通常的委托只支持一对一的委托, 但是在某些场景下, 我们希望有多个被委托者. 这种场景下可以考虑使用多播委托.

多播委托的实现类在XYMulticastDelegate, <https://github.com/uxyheaven/XYQuick/tree/master/XYQuick/event/modules>.
他是copy form XMPP的GCDMulticastDelegate.

每个多播委托的委托者类建议有以下的基本描述

```
// .h
// 多播委托, 建议加上你的协议修饰: -(id <InAppPurchasesServiceProtocol>)multicastDelggate;
- (id)multicastDelggate;
- (void)addDelegate:(id)delegate;
- (void)removeDelegate:(id)delegate;
- (void)removeAllDelegates;
```

实现文件里, 你需要这么写

```
// .m
- (void)addDelegate:(id)delegate
{
    [_multicastDelggate addDelegate:delegate delegateQueue:dispatch_get_main_queue()];
}

- (void)removeDelegate:(id)delegate
{
    [_multicastDelggate removeDelegate:delegate];
}

- (void)removeAllDelegates
{
    [_multicastDelggate removeAllDelegates];
}

```

其他的用法和普通的delegate类似

```
// 协议
@protocol InAppPurchasesServiceProtocol <NSObject>
@optional
/**
 * @brief callback for update product list succeed
 */
-(void)inAppPurchasesService:(InAppPurchasesService *)service didUpdatedProducts:(NSArray *)array;

// 用的时候
[self.multicastDelggate inAppPurchasesService:self didUpdatedProductsFailed:error];

```
## 参考文档
<http://www.jianshu.com/p/b6434c2997d1>
<http://leopard168.blog.163.com/blog/static/168471844201306114533858/>
<https://github.com/robbiehanson/XMPPFramework>
