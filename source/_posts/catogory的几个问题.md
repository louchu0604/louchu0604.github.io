---
title: 关于catogory的几个问题
date: 2018-11-26 00:51:02
tags: objc iOS OC
---
先介绍分类：
分类可以为已经存在的类添加方法，还可以把类的实现分在几个不同的文件里。
Q1：
下面这段话摘自美团点评的文章：
>>>category是无法添加实例变量的（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）
我们可以在objc的源码中看到category的结构，如下：
```objc
typedef struct category_t {
    const char *name;// 类名
    classref_t cls;// 要扩展的类对象
    struct method_list_t *instanceMethods;// 实例方法
    struct method_list_t *classMethods;// 类方法
    struct protocol_list_t *protocols;// 分类实现的协议
    struct property_list_t *instanceProperties;// 分类的属性
} category_t;

```
从结构中可以知道，分类可以添加实例方法、类方法、协议、属性，但是无法添加实例变量。

Q2：
为什么分类方法会覆盖原来的类方法
# 先问是不是，再问为什么
### 首先答案是NO 验证方法很简单，打印出方法列表，会发现原来的类方法还在，所谓的覆盖指的是“分类中的方法在方法查找时会先于原来的类方法被命中”
###为什么呢？
运行时 遍历各个分类中的方法【listA】与原来类的方法【listB】然后组成一个大的方法列表newlist，A在前，B在后，如果listA与listB中都有一个methodA，那么在方法查找时，排在newlist前的分类方法会先被找到，就会让调用者产生“原来的类方法被分类方法覆盖之感”
以此可推，如果在实际应用中我们想要调用“被覆盖的方法”，可以获取类的方法列表，从后往前遍历，找到要调用的方法即可【或者从前往后遍历，获取同名的方法列表，然后取出最后一个】

Q3：

