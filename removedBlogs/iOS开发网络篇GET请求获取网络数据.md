---
title: iOS开发网络篇GET请求获取网络数据
date: 2016-03-11 20:35:11
tags:
---
**这篇主要介绍通过URL获取网络数据**


正好之前做天气应用用到了百度的**免费**接口，所以接下来的例子里用的是百度api，apikey只要有百度账号就可以了  

**一、接口地址、请求参数**
 ```objc
 NSString *httpUrl = @"http://apis.baidu.com/heweather/weather/free";
    NSString *httpArg = @"city=beijing";
    NSString *urlStr = [[NSString alloc]initWithFormat: @"%@?%@", httpUrl, httpArg];
   NSURL *url = [NSURL URLWithString: urlStr];
   ```


**二、请求这个地址， timeoutInterval:10 设置为10s超时：请求时间超过10s会被认为连接不上，连接超时 请求方法**
 ```objc
 NSMutableURLRequest *request = [[NSMutableURLRequest alloc]initWithURL: url cachePolicy: NSURLRequestUseProtocolCachePolicy timeoutInterval: 10];
    [request setHTTPMethod: @"GET"];
    [request addValue: @"f588ddb737e3a0dd74da9dd50b01787f" forHTTPHeaderField: @"apikey"];
    ```
apikey可以在网站上申请


**三、请求**
```objc
[NSURLConnection sendAsynchronousRequest:request queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse *response, NSData *data, NSError *error) {
        if (error) {
            NSLog(@"Httperror: %@%ld", error.localizedDescription, error.code);
        }else{
            NSLog(@"%@",data);
        }
    }];
```
**注意事项：Xcode7需要在plist里加一下内容：App Transport Security Settings 以及Allow Arbitrary Loads**




[示例代码](http://git.oschina.net/Chu_Lou/Example)