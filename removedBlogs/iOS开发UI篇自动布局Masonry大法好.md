---
title: iOS开发UI篇自动布局Masonry大法
date: 2016-03-05 20:37:18
tags:
---
江湖规矩先上链接，[Masonry 源码](https://github.com/Masonry/Masonry)

Masonry是一个轻量级的布局框架，拥有自己的描述语法，在子视图比较多得情况下比手写frame高效简洁，



**一、先创建子视图，并添加到父视图**

```objc
    UIButton *button1 = [UIButton buttonWithType:UIButtonTypeCustom];
    button1.backgroundColor = [[UIColor redColor]colorWithAlphaComponent:0.5f];
   [self.view addSubview:button1];
   ```


    
依次添加其他视图
    
**二、添加约束**

```objc
 [button1 mas_makeConstraints:^(MASConstraintMaker *make) {
//     上边距离父视图上边3个单位
        make.top.equalTo(self.view.mas_top).with.offset(3);
//        左边距离父视图左边3个单位
        make.left.equalTo(self.view.mas_left).with.offset(3);
//        宽度是父视图宽度的一半
        make.width.equalTo(self.view.mas_width).with.multipliedBy(0.5);
//        高度是父视图高度的0.2倍
        make.height.equalTo(self.view.mas_height).with.multipliedBy(0.2);
    }];

   [button2 mas_makeConstraints:^(MASConstraintMaker *make) {
       make.top.equalTo(self.view.mas_top).with.offset(3);       
//       右边距离父视图右边3个单位，-3表示在父视图内，如果是+3的话会跑出父视图外
       make.right.equalTo(self.view.mas_right).with.offset(-3);
//       左边距离button1的右边3个单位
        make.left.equalTo(button1.mas_right).with.offset(3);
       make.height.equalTo(self.view.mas_height).with.multipliedBy(0.2);
          }];

[button3 mas_makeConstraints:^(MASConstraintMaker *make) {
//      上边距离button1的下边3个单位
        make.top.equalTo(button1.mas_bottom).with.offset(3);
        make.left.equalTo(self.view.mas_left).with.offset(3);
//     宽度与button1等宽
        make.width.equalTo(button1.mas_width);
//    高度与button1等高
        make.height.equalTo(button1.mas_height);
    }];
```

三、注意事项

1.  注意区分**with**和**width**，不然会报错
2.  offset参数注意正负

最后效果如图
![](http://ww3.sinaimg.cn/large/62e72542gw1f28dwl6mxzj20h90uomxl.jpg)

[示例代码](http://git.oschina.net/Chu_Lou/Example)