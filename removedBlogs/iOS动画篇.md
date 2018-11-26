---
title: iOS动画篇
date: 2016-04-26 22:06:07
tags:
---

介绍顺便复习一下天气app用到的动画
POP由Facebook出品 ，是一个非常成熟效果丰富的动画引擎，安装方式可以使用CocoaPods，或者在[https://github.com/facebook/pop ](https://github.com/facebook/pop)上下载。
POP默认支持三种动画 也支持自定义动画
* POPBasicAnimation
* POPSpringAnimation
* POPDecayAnimation
* POPCustomAnimation //自定义动画 
添加动画的主要步骤
1. 定义一个animation对象 并指定对应的动画属性
2. 设置初始值（fromValue）和默认值（toValue）（初始值可以不指定 会默认从当前值开始）以及动画时间间隔（duration）
3. 添加到想产生动画的对象上
举例介绍常用的几个动画
1.位移动画
<embed src="http://player.youku.com/player.php/sid/XMTU4MzYxNjY2NA==/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>
指定动画属性
绑定动画
```objc
- (void)startDisplaceAnimation{
    UILabel *redSquare = [[UILabel alloc]initWithFrame:CGRectMake(5, 20, 40, 40)];
    redSquare.backgroundColor = [UIColor redColor];
    [self.view addSubview:redSquare];
    POPBasicAnimation *displaceAni = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerPositionX];
    displaceAni.toValue = @(redSquare.center.x +300);
    displaceAni.beginTime = CACurrentMediaTime() + 5.f;
    displaceAni.duration = 0.4f;
    [redSquare pop_addAnimation:displaceAni forKey:@"position"];
}
```
2.圆环动画
<embed src="http://player.youku.com/player.php/sid/XMTU4MzYxNjQzMg==/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>
初始化圆环，

```objc
 CircleView *circle1 = [CircleView circleViewWithFrame:CGRectMake(0, 0, radius, radius)
                                               lineWidth:2
                                               lineColor:[UIColor grayColor]
                                               clockWise:YES
                                              startAngle:0];
    
    [circle1 buildView];

```
设置timer
```objc
self.timer = [[GCDTimer alloc] initInQueue:[GCDQueue mainQueue]];
    [self.timer event:^{
        
        CGFloat percent        = arc4random() % 100 / 100.f;
        CGFloat anotherPercent = arc4random() % 100 / 100.f;
        
        // 圆圈1动画
        [circle1 strokeEnd:percent animationType:ElasticEaseInOut animated:YES duration:1.f];
        
       
        
    } timeIntervalWithSecs:1.5f];
    
    [self.timer start];
```
3.数值动画
<embed src="http://player.youku.com/player.php/sid/XMTU4MzYxNjEyOA==/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>
初始化label

```objc
 _label               = [[UILabel alloc] initWithFrame:CGRectMake(0, 0, 250, 250)];
    _label.textAlignment = NSTextAlignmentCenter;
    _label.backgroundColor = [[UIColor blackColor]colorWithAlphaComponent:0.1];
    _label.center        = self.view.center;
    [self.view addSubview:_label];
```
设置timer

```objc
self.timer = [[GCDTimer alloc] initInQueue:[GCDQueue mainQueue]];
    [self.timer event:^{
        
        // Start animation.
        [weakSelf configNumberAnimation];
        [weakSelf.numberAnimation startAnimation];
        
    } timeIntervalWithSecs:3.f];
    [self.timer start];
```
4.轨迹动画

<embed src="http://player.youku.com/player.php/sid/XMTU4MzYxNTc2MA==/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>


初始化小方块
```objc
UILabel *redlabel = [[UILabel alloc]initWithFrame:CGRectMake(10, 10, 10, 10)];
    redlabel.backgroundColor = [UIColor redColor];
    [self.view addSubview:redlabel];
```
设置动画开始位置和结束位置
```objc
//    起始点
    self.startPoint = CGPointMake(0, 10);
//    结束点
    self.endPoint = CGPointMake(130, 200);
```
绘制动画轨迹
```objc
UIBezierPath* bezierPath = [UIBezierPath bezierPath];
    
    [bezierPath moveToPoint:self.startPoint];
    [bezierPath addLineToPoint:CGPointMake(0, 20)];
    [bezierPath addLineToPoint:CGPointMake(20, 20)];
    [bezierPath addLineToPoint:CGPointMake(20,40)];
    [bezierPath addLineToPoint:CGPointMake(40, 40)];
    [bezierPath addLineToPoint:CGPointMake(40, 60)];
    [bezierPath addLineToPoint:CGPointMake(60, 60)];
    [bezierPath addLineToPoint:CGPointMake(60, 80)];
    [bezierPath addLineToPoint:self.endPoint];
```


设置动画属性
```objc
 CAKeyframeAnimation *pathAnimation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    pathAnimation.path = [self path].CGPath;
    pathAnimation.duration = 4.f;
    pathAnimation.timingFunction       = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    redlabel.center = self.endPoint;
 ```

为label添加动画
```objc
[redlabel.layer addAnimation:pathAnimation forKey:nil];
```
5.雪花动画
<embed src="http://player.youku.com/player.php/sid/XMTU4MzYxNjg0NA==/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>
这个主要在下雪或者下雨时提供个性化的view
首先要准备粒子素材（一个小圈圈就可以了）
创建雪花粒子的layer
```objc
 self.view.backgroundColor = [UIColor blackColor];
    
    CAEmitterLayer *snowEmitter = [CAEmitterLayer layer];

```
设置发射源的属性

```objc
//    设置发射源位置、尺寸大小，发射模式、发射源形状
//发射源设置在view外面，效果逼真一些
    snowEmitter.emitterPosition = CGPointMake(120,-120);
    snowEmitter.emitterSize = self.view.bounds.size;
    snowEmitter.emitterMode  = kCAEmitterLayerSurface;
    snowEmitter.emitterShape = kCAEmitterLayerLine;
    ```
创建雪花粒子
```objc
//    创建雪花粒子 设置名称
    CAEmitterCell *snowflake = [CAEmitterCell emitterCell];
    snowflake.name = @"snow";
    

```
设置雪花粒子速度属性

```objc
//     粒子参数的速度乘数因子
    snowflake.birthRate = 20.0;
    snowflake.lifetime  = 120.0;
    
// 粒子速度
    snowflake.velocity =10.0;
    
// 粒子的速度范围
    snowflake.velocityRange = 10;
    
// 粒子y方向的加速度分量
    snowflake.yAcceleration = 2;
    
// 周围发射角度
    snowflake.emissionRange = 0.5 * M_PI;
    
// 子旋转角度范围
    snowflake.spinRange = 0.25 * M_PI;
    snowflake.contents  = (id)[[UIImage imageNamed:@"snow"] CGImage];

```
设置颜色

```objc
//设置雪花为纯白
    snowflake.color = [[UIColor whiteColor]CGColor];
    
    snowflake.scaleRange = 0.6f;
    snowflake.scale      = 0.7f;
    
    snowEmitter.shadowOpacity = 1.0;
    snowEmitter.shadowRadius  = 0.0;
    snowEmitter.shadowOffset  = CGSizeMake(0.0, 0.0);
    // 粒子边缘的颜色
    snowEmitter.shadowColor  = [[UIColor whiteColor] CGColor];
```
添加粒子到layer

```objc
 // 添加粒子
    snowEmitter.emitterCells = @[snowflake];
```
添加layer到view中

```objc
// 将粒子Layer添加进图层中
    [self.view.layer addSublayer:snowEmitter];
```




