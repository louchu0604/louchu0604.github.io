---
title: iOS图片加载框架SDWebImage
date: 2016-03-27 15:59:12
tags:
---
在做天气应用的时候希望有一个新功能就是背景图片可以更换，所以就有了下面的方法。
微软的Bing首页每天都会更新漂亮的图片，下面讲一下获取Bing首页图片的方式。
```objc
NSError *err;
    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:@"http://cn.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1"]];
    NSData *reponse = [NSURLConnection sendSynchronousRequest:request returningResponse:nil error:nil];
    NSDictionary *picDic = [NSJSONSerialization JSONObjectWithData:reponse options:NSJSONReadingMutableContainers error:&err];
   
    NSDictionary *picc = [picDic objectForKey:@"images"];
    NSArray *arr = picc;
    NSArray *arr1 = arr[0];
    NSDictionary *dic = arr1;
    NSString *picStr = [dic objectForKey:@"url"];
    NSURL *picURL = [NSURL URLWithString:picStr];
    UIImageView *picView = [[UIImageView alloc]initWithFrame:self.view.bounds];
    [picView sd_setImageWithURL:picURL];
    [self.view addSubview:picView];
```

然而，用这种方式加载出来图片的填充方式是UIViewContentModeScaleAspectFit，页面会留很多空白，我并不太期待这样的效果。
所以改成了下面的这个：
```objc
 NSURL *picURL = [NSURL URLWithString:picStr];
    UIImageView *picView = [[UIImageView alloc]initWithFrame:self.view.bounds];
    picView.contentMode = UIViewContentModeScaleAspectFill;
    //    NSData *data = [NSData dataWithContentsOfURL:picURL];
    //    picView.image = [UIImage imageWithData:data];
    [picView sd_setImageWithURL:picURL];
    //    NSLog(@"%f-------%f",picView.image.size.width,picView.image.size.height);
    
    [self.view addSubview:picView];
```