---
title: iOS中的delegated的用法和规范
date: 2017-01-18 17:28:33
tags:
categories: iOS
---

# NSObject 的实现分析
iOS 的 NSObject 类没有开源, 但是呢 runtime开源了,里面有个类 Object 看接口和NSObject差不多,下面我就对着 Object 的代码来分析下 NSObject
runtime代码在<http://opensource.apple.com/tarballs/objc4/objc4-493.9.tar.gz>下载,
Object在<Object.h>, 这里的文件夹写着Obsolete, 呃.
<!-- more -->

# NSObject 的实现分析

本次分析暂时告一段落

转载请注名出处 [http://blog.csdn.net/uxyheaven](http://blog.csdn.net/uxyheaven/article/details/38120335)

iOS 的 NSObject 类没有开源, 但是呢 runtime开源了,里面有个类 Object 看接口和NSObject差不多,下面我就对着 Object 的代码来分析下 NSObject

runtime代码在<http://opensource.apple.com/tarballs/objc4/objc4-493.9.tar.gz>下载,
Object在<Object.h>, 这里的文件夹写着Obsolete, 呃.

### 属性
#### isa
是一个指向Class的指针,具体请看这篇文章
[Objective-C objc_class 介绍](http://blog.csdn.net/uxyheaven/article/details/38113901)
### 方法
#### class
实例方法返回的是isa指针, 类方法返回的是本身

代码实现如下:

```
- class
{
	return (id)isa;
}

+ class
{
	return self;
}
```

#### superclass
返回父类

代码实现如下:

```
+ superclass
{
	return class_getSuperclass((Class)self);
}

- superclass
{
	return class_getSuperclass(isa);
}
```

调用的是runtime中的class_getSuperclass方法, 跟踪到最后实例方法返回的是isa->superclass,类方法返回的是self->superclass

```
static class_t *
getSuperclass(class_t *cls)
{
    if (!cls) return NULL;
    return cls->superclass;
}
```
#### isEqual
就是直接比较

```
- (BOOL)isEqual:anObject
{
	return anObject == self;
}
```
#### isMemberOf:
```
- (BOOL)isMemberOf:aClass
{
	return isa == (Class)aClass;
}
```
看代码可以得知是通过比较实例对象的isa是否和 传过来的[类 Class] 一样来判断的.而实例对象的isa确实就是指着实例对象的类的.

#### isKindOf:
```
- (BOOL)isKindOf:aClass
{
	register Class cls;
	for (cls = isa; cls; cls = class_getSuperclass(cls))
		if (cls == (Class)aClass)
			return YES;
	return NO;
}

// class_getSuperclass 展开后如下
static class_t *
getSuperclass(class_t *cls)
{
    if (!cls) return NULL;
    return cls->superclass;
}
```
代码思路也很好理解,如果自己的isa等于aClass(aClass的父类,此处循环)就返回YES,否则返回NO

#### init
```
- init
{
    return self;
}
```
没什么好说的

#### alloc
```
+ alloc
{
	return (*_zoneAlloc)((Class)self, 0, malloc_default_zone());
}
```
这里有一个函数指针和一个结构体,我们跟进去看

```
id (*_zoneAlloc)(Class, size_t, void *) = _class_createInstanceFromZone;

PRIVATE_EXTERN id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone)
{
    id obj;
    size_t size;

    // Can't create something for nothing
    if (!cls) return nil;

    // Allocate and initialize
    size = _class_getInstanceSize(cls) + extraBytes;

    // CF requires all objects be at least 16 bytes.
    if (size < 16) size = 16;

#if SUPPORT_GC
    if (UseGC) {
        obj = (id)auto_zone_allocate_object(gc_zone, size,
                                            AUTO_OBJECT_SCANNED, 0, 1);
    } else
#endif
    if (zone) {
        obj = (id)malloc_zone_calloc (zone, 1, size);
    } else {
        obj = (id)calloc(1, size);
    }
    if (!obj) return nil;

    obj->isa = cls;

    if (_class_hasCxxStructors(cls)) {
        obj = _objc_constructOrFree(cls, obj);
    }

    return obj;
}

```
上面那段代码的作用是

1. 得到这个类占用多少空间,最小占16 bytes
2. 然后就给这个实例分配多少空间, 如果失败的话就返回nil
3. 把这个实例的isa设置成这个类对象
4. 如果cls的info设置了get属性就用cls这个类在obj这个空间去构造一个实例,跟进去是

```
static BOOL object_cxxConstructFromClass(id obj, Class cls)
{
    id (*ctor)(id);
    Class supercls;

    // Stop if neither this class nor any superclass has ctors.
    if (!_class_hasCxxStructors(cls)) return YES;  // no ctor - ok

    supercls = _class_getSuperclass(cls);

    // Call superclasses' ctors first, if any.
    if (supercls) {
        BOOL ok = object_cxxConstructFromClass(obj, supercls);
        if (!ok) return NO;  // some superclass's ctor failed - give up
    }

    // Find this class's ctor, if any.
    ctor = (id(*)(id))lookupMethodInClassAndLoadCache(cls, SEL_cxx_construct);
    if (ctor == (id(*)(id))&_objc_msgForward_internal) return YES;  // no ctor - ok

    // Call this class's ctor.
    if (PrintCxxCtors) {
        _objc_inform("CXX: calling C++ constructors for class %s", _class_getName(cls));
    }
    if ((*ctor)(obj)) return YES;  // ctor called and succeeded - ok

    // This class's ctor was called and failed.
    // Call superclasses's dtors to clean up.
    if (supercls) object_cxxDestructFromClass(obj, supercls);
    return NO;
}
```
大意是,先看自己有没有父类,有就递归调用自己,然后给自己添加方法,然后添加类别



#### new
```
+ new
{
	id newObject = (*_alloc)((Class)self, 0);
	Class metaClass = self->isa;
	if (class_getVersion(metaClass) > 1)
	    return [newObject init];
	else
	    return newObject;
}
```
跟进去看一下, 发现是和 alloc差不多

```
id (*_alloc)(Class, size_t) = _class_createInstance;

static id _class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone (cls, extraBytes, NULL);
}
```

#### free

```
- free
{
	return (*_dealloc)(self);
}

+ free
{
	return nil;
}
```

跟进去看一下

```
static id
_object_dispose(id anObject)
{
    if (anObject==nil) return nil;

    objc_destructInstance(anObject);

#if SUPPORT_GC
    if (UseGC) {
        auto_zone_retain(gc_zone, anObject); // gc free expects rc==1
    } else
#endif
    {
        // only clobber isa for non-gc
        anObject->isa = _objc_getFreedObjectClass ();
    }
    free(anObject);
    return nil;
}

void *objc_destructInstance(id obj)
{
    if (obj) {
        Class isa = _object_getClass(obj);

        if (_class_hasCxxStructors(isa)) {
            object_cxxDestruct(obj);
        }

        if (_class_instancesHaveAssociatedObjects(isa)) {
            _object_remove_assocations(obj);
        }

        if (!UseGC) objc_clear_deallocating(obj);
    }

    return obj;
}
```

1. 执行一个叫object_cxxDestruct的东西干了点什么事(沿着继承链逐层向上搜寻SEL_cxx_destruct这个selector, 找到函数实现(void (*)(id)(函数指针)并执行)
2. 执行_object_remove_assocations去除和这个对象关联的对象
3. 执行objc_clear_deallocating，清空引用计数表并清除弱引用表，将所有weak引用指nil

#### respondsTo:
是查找有没有实现某个方法

```
- (BOOL)respondsTo:(SEL)aSelector
{
	return class_respondsToMethod(isa, aSelector);
}

BOOL class_respondsToMethod(Class cls, SEL sel)
{
    OBJC_WARN_DEPRECATED;

    return class_respondsToSelector(cls, sel);
}

BOOL class_respondsToSelector(Class cls, SEL sel)
{
    IMP imp;

    if (!sel  ||  !cls) return NO;

    // Avoids +initialize because it historically did so.
    // We're not returning a callable IMP anyway.
    imp = lookUpMethod(cls, sel, NO/*initialize*/, YES/*cache*/);
    return (imp != (IMP)_objc_msgForward_internal) ? YES : NO;
}
```
#### perform:
perform是发送消息到指定的接收器并返回值, 下面是代码:

```
- perform:(SEL)aSelector
{
	if (aSelector)
		return objc_msgSend(self, aSelector);
	else
		return [self error:_errBadSel, sel_getName(_cmd), aSelector];
}
```
原来就是objc_msgSend这玩意.objc_msgSend实现有很多个版本, 大体逻辑应该差不多, 首先在找缓存,找到就跳转过去,找不到就在Class的方法列表里找方法,  如果还是没找到就转发.

下的是arm下的代码

```
ENTRY objc_msgSend
# check whether receiver is nil
	teq     a1, #0
	itt	eq
	moveq   a2, #0
	bxeq    lr

# save registers and load receiver's class for CacheLookup
	stmfd   sp!, {a4,v1}
	ldr     v1, [a1, #ISA]

# receiver is non-nil: search the cache
	CacheLookup a2, v1, LMsgSendCacheMiss

# cache hit (imp in ip) and CacheLookup returns with nonstret (eq) set, restore registers and call
	ldmfd   sp!, {a4,v1}
	bx      ip

# cache miss: go search the method lists
LMsgSendCacheMiss:
	ldmfd	sp!, {a4,v1}
	b	_objc_msgSend_uncached

LMsgSendExit:
	END_ENTRY objc_msgSend


	STATIC_ENTRY objc_msgSend_uncached

# Push stack frame
	stmfd	sp!, {a1-a4,r7,lr}
	add     r7, sp, #16

# Load class and selector
	ldr	a1, [a1, #ISA]		/* class = receiver->isa  */
	# MOVE	a2, a2			/* selector already in a2 */

# Do the lookup
	MI_CALL_EXTERNAL(__class_lookupMethodAndLoadCache)
	MOVE    ip, a1

# Prep for forwarding, Pop stack frame and call imp
	teq	v1, v1		/* set nonstret (eq) */
	ldmfd	sp!, {a1-a4,r7,lr}
	bx	ip
```
#### conformsTo:
返回是否遵循了某个协议

```
- (BOOL) conformsTo: (Protocol *)aProtocolObj
{
  return [(id)isa conformsTo:aProtocolObj];
}

+ (BOOL) conformsTo: (Protocol *)aProtocolObj
{
  Class class;
  for (class = self; class; class = class_getSuperclass(class))
    {
      if (class_conformsToProtocol(class, aProtocolObj)) return YES;
    }
  return NO;
}
```

最终用的是class_conformsToProtocol, 返回一个布尔值,表示一个类是否符合给定的协议。

class_conformsToProtocol的实如下

```
BOOL class_conformsToProtocol(Class cls_gen, Protocol *proto_gen)
{
    struct old_class *cls = oldcls(cls_gen);
    struct old_protocol *proto = oldprotocol(proto_gen);

    if (!cls_gen) return NO;
    if (!proto) return NO;

    if (cls->isa->version >= 3) {
        struct old_protocol_list *list;
        for (list = cls->protocols; list != NULL; list = list->next) {
            int i;
            for (i = 0; i < list->count; i++) {
                if (list->list[i] == proto) return YES;
                if (protocol_conformsToProtocol((Protocol *)list->list[i], proto_gen)) return YES;
            }
            if (cls->isa->version <= 4) break;
        }
    }
    return NO;
}
```
可以看到是在cls->protocols里面找.protocols 是协议的数组

#### copy
浅拷贝

```
- copy
{
	return [self copyFromZone: [self zone]];
}

// 返回指定区域的指针
- (void *)zone
{
	void *z = malloc_zone_from_ptr(self);
	return z ? z : malloc_default_zone();
}

- copyFromZone:(void *)z
{
	return (*_zoneCopy)(self, 0, z);
}

id (*_zoneCopy)(id, size_t, void *) = _object_copyFromZone;

static id _object_copyFromZone(id oldObj, size_t extraBytes, void *zone)
{
    id obj;
    size_t size;

    if (!oldObj) return nil;

	// 用旧对象的isa生成一个新的对象的空间
    obj = (*_zoneAlloc)(oldObj->isa, extraBytes, zone);
    size = _class_getInstanceSize(oldObj->isa) + extraBytes;

    // fixme need C++ copy constructor
    // 把旧对象的内存拷贝到新对象
    objc_memmove_collectable(obj, oldObj, size);

#if SUPPORT_GC
    if (UseGC) gc_fixup_weakreferences(obj, oldObj);
#endif

 }
```
