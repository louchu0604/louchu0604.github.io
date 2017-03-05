---
title: iOS开发UI篇UIButton
date: 2016-03-06 20:37:21
tags:
---
**用代码的方式创建UIButton，较storyboard或者xib而言，其优点是便于修改，容易维护**
1.先定义一个button
```objc
UIButton *button1 = [UIButton buttonWithType:UIButtonTypeRoundedRect];
```

button的类型主要有六种：

1.    UIButtonTypeCustom = 0, 自定义风格
2.    UIButtonTypeRoundedRect, 圆角矩形
3.    UIButtonTypeDetailDisclosure, 蓝色小箭头按钮，主要做详细说明用
4.    UIButtonTypeInfoLight, 亮色感叹号
5.    UIButtonTypeInfoDark, 暗色感叹号
6.    UIButtonTypeContactAdd, 十字加号按钮

使用第二种圆角类型还需要设置圆角半径
```objc
[button1.layer setMasksToBounds:YES];
[button1.layer setCornerRaius:4.f];  //半径设为4毫米
```

2.设置frame 两种方式都可以
```objc
  button1.frame = CGRectMake(50, 50, 250, 50);

 [button1 setFrame:CGRectMake(50, 50, 250, 50)];
```

3.设置背景颜色
```objc
button1.backgroundColor = [UIColor redColor];
[button1 setBackgroundColor:[UIColor redColor]];
```

4.设置背景图片
```objc
[button1 setImage:@"这里写图片名称" forState:UIControlStateNormal];
```
  
forState参数：定义按钮的文字或者图片在什么状态下出现

1. UIControlStateNormal = 0, 常规状态显现
2. UIControlStateHighlighted = 1 << 0, 高亮状态显现
3. UIControlStateDisabled = 1 << 1, 禁用的状态才会显现
4. UIControlStateSelected = 1 << 2, 选中状态
5. UIControlStateApplication = 0x00FF0000, 当应用程序标志时
6. UIControlStateReserved = 0xFF000000 为内部框架预留，可以不管

5.设置标题及标题颜色
```objc
[button1 setTitle:@"这里写按钮上的文字" forState:UIControlStateNormal];
[button1 setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
```

6.设置按钮按下会发光
```objc
button1.showsTouchWhenHighlighted = YES;
```
7.添加点击事件
```objc
[button1 addTarget:self action:@selector(buttonTouch) forControlEvents:UIControlEventTouchUpInside];
```
```objc
- (void)buttonTouch{//点击按钮时输出button is touched
    NSLog(@"button is touched");
}
```
8.添加长按钮事件
```objc
 UILongPressGestureRecognizer *longPress = [[UILongPressGestureRecognizer alloc]initWithTarget:self action:@selector(longPressDo)];
    [button1 addGestureRecognizer:longPress];
- (void)longPressDo{
    NSLog(@"button is longpressed");
}
```

[示例代码](http://git.oschina.net/Chu_Lou/Example)
