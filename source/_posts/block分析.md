---
title: block内部结构
date: 2017-01-18 17:28:33
tags:
categories: iOS
---

## block内部结构

本文主页对block内部结构进行分析

<!--more-->

我们先写一个block

``` Objective-C
void exampleBlock() {
	// NSConcreteStackBlock
    int a = 1;
    __block int b = 2;
    int(^blockTest0)(int c) = ^(int c){
        return a + b + c;
    };
    int c = 3;
    blockTest0(c);

    // NSConcreteGlobalBlock
    void(^blockTest2)(void) = ^(void){
        ;
    };
    blockTest2();
}
```

用clang转成c分析下

```
clang -rewrite-objc block.c
```

可以看到他们的定义是

```
struct __exampleBlock_block_impl_0 {
  struct __block_impl impl;
  struct __exampleBlock_block_desc_0* Desc;
  int a;
  __Block_byref_b_0 *b; // by ref
  __exampleBlock_block_impl_0(void *fp, struct __exampleBlock_block_desc_0 *desc, int _a, __Block_byref_b_0 *_b, int flags=0) : a(_a), b(_b->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

// __block_impl
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

// __exampleBlock_block_desc_0
struct __exampleBlock_block_impl_0 {
  struct __block_impl impl;
  struct __exampleBlock_block_desc_0* Desc;
  int a;
  __Block_byref_b_0 *b; // by ref
  __exampleBlock_block_impl_0(void *fp, struct __exampleBlock_block_desc_0 *desc, int _a, __Block_byref_b_0 *_b, int flags=0) : a(_a), b(_b->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

// __exampleBlock_block_impl_2
struct __exampleBlock_block_impl_2 {
  struct __block_impl impl;
  struct __exampleBlock_block_desc_2* Desc;
  __exampleBlock_block_impl_2(void *fp, struct __exampleBlock_block_desc_2 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

初始化和执行代码

```
void exampleBlock() {
	// blockTest0
    int a = 1;
    __attribute__((__blocks__(byref))) __Block_byref_b_0 b = {(void*)0,(__Block_byref_b_0 *)&b, 0, sizeof(__Block_byref_b_0), 2};

    int(*blockTest0)(int c) = (int (*)(int))&__exampleBlock_block_impl_0((void *)__exampleBlock_block_func_0, &__exampleBlock_block_desc_0_DATA, a, (__Block_byref_b_0 *)&b, 570425344);

    int c = 3;
    ((int (*)(__block_impl *, int))((__block_impl *)blockTest0)->FuncPtr)((__block_impl *)blockTest0, c);

	// blockTest2
    void(*blockTest2)(void) = (void (*)())&__exampleBlock_block_impl_2((void *)__exampleBlock_block_func_2, &__exampleBlock_block_desc_2_DATA);
    ((void (*)(__block_impl *))((__block_impl *)blockTest2)->FuncPtr)((__block_impl *)blockTest2);
}
```


我们先看看blockTest2,它是由 结构体impl, 结构体Desc, 构造方法__exampleBlock_block_impl_2() 组成
展开后是

* *isa 指向该实例对象(代码里是_NSConcreteStackBlock,其实应该是_NSConcreteGlobalBlock)
* Flags 用于按bit位表示一些block的附加信息
* reserved 保留变量
* *FuncPtr 函数指针,指向具体的block实现的函数调用地址(代码里是__exampleBlock_block_func_2)

```
static void __exampleBlock_block_func_2(struct __exampleBlock_block_impl_2 *__cself) {
        ;
}
```

* size_t reserved 这个传进来的是0
* Block_size 结构体的大小



```
static struct __exampleBlock_block_desc_2 {
  size_t reserved;
  size_t Block_size;
} __exampleBlock_block_desc_2_DATA = { 0, sizeof(struct __exampleBlock_block_impl_2)};
```
---
然后我们在看blockTest,它比blockTest2多了2个变量a, b

* int a; 外部变量a,

* __Block_byref_b_0 *b; 加了block修饰的b, 传的是 __Block_byref_b_0


在生成的初始化代码中则多了3个传入值

```
    int(*blockTest0)(int c) = (int (*)(int))&__exampleBlock_block_impl_0((void *)__exampleBlock_block_func_0, &__exampleBlock_block_desc_0_DATA, a, (__Block_byref_b_0 *)&b, 570425344);
```
* a是这个是直接传值进去,然后被复制给 a

* b是传的地址, 是把 __Block_byref_b_0 赋值给 b


__Block_byref_b_0这个结构体是

```
struct __Block_byref_b_0 {
  void *__isa;
__Block_byref_b_0 *__forwarding;
 int __flags;
 int __size;
 int b;
};
```
 __forwarding 是一个指向自己的指针.

__Block_byref_b_0 的初始化代码如下:

```
__attribute__((__blocks__(byref))) __Block_byref_b_0 b = {(void*)0,(__Block_byref_b_0 *)&b, 0, sizeof(__Block_byref_b_0), 2};
```

 我们可以看出a是直接复制进去的,b是被转到了一个结构体里,然后吧这个结构体的指针传进去,所以block不能修改a,能修改b.

* 570425344 这个应该是传给Flags

blockTest0的Desc和blockTest2也有所不同,展开后是

```
static struct __exampleBlock_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __exampleBlock_block_impl_0*, struct __exampleBlock_block_impl_0*);
  void (*dispose)(struct __exampleBlock_block_impl_0*);
} __exampleBlock_block_desc_0_DATA = { 0, sizeof(struct __exampleBlock_block_impl_0), __exampleBlock_block_copy_0, __exampleBlock_block_dispose_0};
```

多了2个函数指针copy, dispose,对于在调用前后修改相应变量的引用计数, 分别指向

```
static void __exampleBlock_block_copy_0(struct __exampleBlock_block_impl_0*dst, struct __exampleBlock_block_impl_0*src) {_Block_object_assign((void*)&dst->b, (void*)src->b, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __exampleBlock_block_dispose_0(struct __exampleBlock_block_impl_0*src) {_Block_object_dispose((void*)src->b, 8/*BLOCK_FIELD_IS_BYREF*/);}

```

 再来看下blockTest0的*FuncPtr

``` Objective-C
 static int __exampleBlock_block_func_0(struct __exampleBlock_block_impl_0 *__cself, int c) {
  __Block_byref_b_0 *b = __cself->b; // bound by ref
  int a = __cself->a; // bound by copy

        return a + (b->__forwarding->b) + c;
    }
```
可以看出a用的是拷贝进来的a, b用的是结构体里的b
