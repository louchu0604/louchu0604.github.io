---
title: MBProgressHUD初体验
date: 2016-03-03 18:55:26
tags:
---
**下面介绍用开源的MBProgressHUD类实现等待框**
一、下载MBProgressHUD类文件并导入工程。下载地址：[MBProgressHUD](http://github.com/matej/MBProgressHUD)
二、示例代码
```objc

#import "ViewController.h"
#import "MBProgressHUD.h"

@interface ViewController ()<MBProgressHUDDelegate>
{
    MBProgressHUD *_progressView;
}


@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
//    添加按钮
    UIButton *button1 = [[UIButton alloc]initWithFrame:CGRectMake(50, 150, 150, 50)];
    [button1 setTitle:@"弹出提示框" forState:UIControlStateNormal];
    button1.backgroundColor = [UIColor greenColor];
    button1.titleLabel.font = [UIFont systemFontOfSize:20.f];
    button1.titleLabel.textColor = [UIColor blackColor];
    [self.view addSubview:button1];
//   为按钮添加点击事件
    [button1 addTarget:self action:@selector(progressShow) forControlEvents:UIControlEventTouchUpInside];
   
    
    
//    
    _progressView = [[MBProgressHUD alloc]initWithView:self.view];
    [self.view addSubview:_progressView];
    [self.view bringSubviewToFront:_progressView];
    _progressView.delegate = self;
    _progressView.labelText = @"我是提示框";//提示框显示文字
    
    
}

- (void)progressShow{
//    点击事件，弹出提示框
    [_progressView show:YES];
//    5秒时候隐藏提示框
    [_progressView hide:YES afterDelay:5.f];
    
}


- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

@end

```


[示例代码](http://git.oschina.net/Chu_Lou/Example)


