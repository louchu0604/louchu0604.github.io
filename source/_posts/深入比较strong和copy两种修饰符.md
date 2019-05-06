---
title: 深入比较strong和copy两种修饰符
date: 2018-11-26 00:48:10
tags:objc
---
strong和copy是property的两个基本修饰符，在oc中，一般情况下用来修饰对象。
strong：property的默认修饰符。
copy：一般用来修饰拥有可变类型的对象。比如NSString NSDictionary NSArray.
在学习oc的过程中，大多数的资料里都提到了以上两点，却没有详细说明这么做的原因。
本文旨在oc底层实现的基础上探讨以上两点的原因和合理性。

一、strong和copy的不同、以及为什么NSString等对象一般用copy来修饰？
对象使用strong和copy修饰后，都可以让对象的RC＋1。其不同点可以看一下代码。
```objc

@property (nonatomic,strong) id objStrong;
@property (nonatomic,copy)   id objCopy;

id obj = [[NSMutableDictionary alloc] init];
[obj setValue:@"hello" forKey:@"objValue"];
self.objStrong = obj;
self.objCopy = [obj copy];
NSLog(@"before obj changed---objStrong:%@  objCopy:%@",[self.objStrong valueForKey:@"objValue"],[self.objCopy valueForKey:@"objValue"]);

[obj setValue:@"changed" forKey:@"objValue"];
NSLog(@"after obj changed---objStrong:%@  objCopy:%@",[self.objStrong valueForKey:@"objValue"],[self.objCopy valueForKey:@"objValue"]);

```
可以来查看一下输出
```objc
before obj changed---objStrong:hello  objCopy:hello
after obj changed---objStrong:changed  objCopy:hello

```
可以看到copy修饰的对象在obj改变前后没有发生变化，而strong修饰的对象则相反。
可以肯定的是，objStrong和objCopy指向的不是同一个对象。
打印对象地址来验证一下：
```objc
NSLog(@"\rcompare address \robj:%p \robjStrong:%p \robjCopy:%p",obj,self.objStrong,self.objCopy);
```
查看输出：
```objc
compare address 
obj:0x6000018d9180 
objStrong:0x6000018d9180 
objCopy:0x6000018d9160
```
可以看到指向的是堆中两块不同的内存,其中objStrong指向的内存地址与obj的相同。
换句话说，strong修饰的属性没有开辟新的内存空间。而copy修饰的属性开辟了新的内存空间。
在属性不希望被外部修改的时候，我们可以用copy来修饰。原因如上例子所述。


二、从源码看strong的底层原理
（备注 我看的版本是objc4-723）
在NSObject.mm line294中可以看到以下代码
```objc
objc_storeStrong(id *location, id obj)
{
    id prev = *location;
    if (obj == prev) {
        return;
    }
    objc_retain(obj);
    *location = obj;
    objc_release(prev);
}
```
首先判断和之前的引用是否相同
若不同，往下执行，retain obj并释放之前的引用（旧的引用对象rc--）


三、copy 
在objc-runtime-new.mm line 6219
```objc
_object_copyFromZone(id oldObj, size_t extraBytes, void *zone)
{
    if (!oldObj) return nil;
    if (oldObj->isTaggedPointer()) return oldObj;

    // fixme this doesn't handle C++ ivars correctly (#4619414)

    Class cls = oldObj->ISA();
    size_t size;
    id obj = _class_createInstanceFromZone(cls, extraBytes, zone, false, &size);
    if (!obj) return nil;

    // Copy everything except the isa, which was already set above.
    uint8_t *copyDst = (uint8_t *)obj + sizeof(Class);
    uint8_t *copySrc = (uint8_t *)oldObj + sizeof(Class);
    size_t copySize = size - sizeof(Class);
    memmove(copyDst, copySrc, copySize);

    fixupCopiedIvars(obj, oldObj);

    return obj;
}
```
可以看到这句
```objc
id obj = _class_createInstanceFromZone(cls, extraBytes, zone, false, &size);

```


