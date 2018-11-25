---
title: 天气应用开发过程及调试
date: 2016-03-22 23:39:39
tags:
---
该应用地址[Biuweather](https://itunes.apple.com/us/app/biuweather/id1095115168?l=zh&ls=1&mt=8)


![](http://ww3.sinaimg.cn/large/0060lm7Tgw1f3bhatduvaj30jz0zk799.jpg)
![](http://ww1.sinaimg.cn/large/0060lm7Tgw1f3bhasnldwj30jz0zk0y2.jpg)
一、天气字体安装及使用
1.将Weather&Time.ttf拖到项目中 （鼠标右键单机选择Add Files To "你的项目"）
2.plist添加 Fonts provided by application 在此字段下添加item，将天气字体的文件名填进去
3.最重要的，Build Phases ->Copy Bundle Resources下添加相应item，否则不起作用
4.使用，下面会举例子


二、UI层
UI分两层，
第一层 ：加载背景图片，这里用到的是Bing搜索首页的每日一图，下面会介绍 
第二层：UI层主要的部分用UITableView实现，将实况天气展示在UItableView的tableHeaderView中，
逐时天气预报和一周天气预报展示在UITableViewCell中
tableHeaderView
1.初始化
```objc
self.tableView = [[UITableView alloc]init];
    
    self.tableView.backgroundColor = [UIColor clearColor];
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    self.tableView.separatorColor = [UIColor colorWithWhite:1 alpha:0.2];
    self.tableView.pagingEnabled = YES;
    [self.view addSubview:self.tableView];
     self.header = [[UIView alloc]initWithFrame:[UIScreen mainScreen].bounds];
    self.header.backgroundColor = [UIColor clearColor];
    
    self.tableView.tableHeaderView = self.header;
```

2.添加label，
```objc

self.currentLabel = [[UILabel alloc]initWithFrame:CGRectMake(20, Height-180, Width - 40, 110)];
    self.currentLabel.backgroundColor = [UIColor clearColor];
    self.currentLabel.textColor = [UIColor whiteColor];
    self.currentLabel.alpha = 0;
    self.currentLabel.font =[UIFont fontWithName:numberfont size:120];
    
    self.maxMinLabel = [[UILabel alloc]initWithFrame:CGRectMake(25, Height - 70, Width - 40, 40)];
    self.maxMinLabel.backgroundColor = [UIColor clearColor];
    self.maxMinLabel.textColor = [UIColor whiteColor];
    self.maxMinLabel.alpha = 0;
    self.maxMinLabel.textAlignment = NSTextAlignmentLeft;
    self.maxMinLabel.font = [UIFont fontWithName:numberfont size:28];
    
    self.cityLabel = [[UILabel alloc]initWithFrame:CGRectMake(0, 30, Width, 30)];
    self.cityLabel.backgroundColor = [UIColor clearColor];
    self.cityLabel.textColor = [UIColor whiteColor];
    self.cityLabel.alpha = 0;
    self.cityLabel.font = [UIFont systemFontOfSize:18];
    self.cityLabel.textAlignment = NSTextAlignmentCenter;
    
    self.weatherDescriLabel = [[UILabel alloc]initWithFrame:CGRectMake(25, Height - 210, 40, 30)];
    self.weatherDescriLabel.backgroundColor = [UIColor clearColor];
    self.weatherDescriLabel.textColor = [UIColor whiteColor];
    self.weatherDescriLabel.alpha = 0;
    self.weatherDescriLabel.font = [UIFont systemFontOfSize:18];
    self.weatherDescriLabel.textAlignment = NSTextAlignmentRight;
    
    self.weatherIconLabel = [[UILabel alloc]initWithFrame:CGRectMake(90, Height - 210, 60, 30)];
    self.weatherIconLabel.backgroundColor = [UIColor clearColor];
    self.weatherIconLabel.textColor = [UIColor whiteColor];
    self.weatherIconLabel.alpha = 0;
    self.weatherIconLabel.textAlignment = NSTextAlignmentCenter;
    self.weatherIconLabel.font = [UIFont fontWithName:WEATHER_TIME size:28];
    
    [self.header addSubview:self.currentLabel];
    [self.header addSubview:self.maxMinLabel];
    [self.header addSubview:self.cityLabel];
    [self.header addSubview:self.weatherDescriLabel];
    [self.header addSubview:self.weatherIconLabel];

```
3.label信息
```objc
self.cityLabel.text =self.cityCNName;
    NSLog(@"-----------%@",self.cityLabel.text);
    self.currentLabel.text = [NSString stringWithFormat:@"%@°",self.getWeatherData.weatherData.today.temp];
    self.currentLabel.text = self.getWeatherData.weatherData.today.temp;
    self.maxMinLabel.text = [NSString stringWithFormat:@"%@°/%@°", self.getWeatherData.weatherData.week.maxTmp,self.getWeatherData.weatherData.week.minTmp];
    self.weatherDescriLabel.text = self.getWeatherData.weatherData.today.weatherTxt;
    self.weatherIconLabel.text = [WeatherNumberMeaningTransform fontTextWeatherNumber:self.getWeatherData.weatherData.today.weatherCode];


```

4.label里的文字慢慢显示
```objc
  [ UIView animateWithDuration:1.0 animations:^{
       self.cityLabel.alpha = 1.0;
    } completion:^(BOOL finished) {
        NSLog(@"citylabel");
    }];
    
    [ UIView animateWithDuration:1.5 animations:^{
        self.weatherIconLabel.alpha = 1.0;
        self.weatherDescriLabel.alpha = 1.0;
    } completion:^(BOOL finished) {
        NSLog(@"weather");
    }];
    [ UIView animateWithDuration:1.8 animations:^{
        self.currentLabel.alpha = 1.0;
    } completion:^(BOOL finished) {
        NSLog(@"currenttmp");
    }];
    [ UIView animateWithDuration:2.0 animations:^{
        self.maxMinLabel.alpha = 1.0;
    } completion:^(BOOL finished) {
        NSLog(@"maxmin");
    }];


```



UITableViewCell
1.表格分两个section，放逐时预报和一周预报
```objc
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
{
    return 2;
}
```
2.每个section行数，逐时预报的count会变化，所以设置成数组里的count，一周天气预报的count不变，所以设置成7		
```objc
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    if (section == 0) {
        return self.getWeatherData.weatherData.hourly.hourlyFor.count;
    }
    return 7;
}

```	
3.每行展示的信息，由2知，第一组每行展示的是逐时天气预报，第二组每行展示的是一周天气预报，

```objc

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{

static NSString *TableSampleIdentifier = @"TableSampleIdentifier";
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:TableSampleIdentifier];
    cell.backgroundColor = [UIColor clearColor];
    //    如果如果没有多余单元，则需要创建新的单元
    if (cell == nil) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleValue1 reuseIdentifier:TableSampleIdentifier];
        cell.backgroundColor = [UIColor clearColor];
    }
    
    else {
        while ([cell.contentView.subviews lastObject ]!=nil) {
            [(UIView*)[cell.contentView.subviews lastObject]removeFromSuperview];
        }
    }
    if (indexPath.section == 1){
    
     //TODO hourlyForecast 
         
    }else{
    
     //TODO dailyForecast
     
      }
    NSUInteger row = [indexPath row];
     return cell;
}

```
TODO那里写布局，
三、数据层介绍
数据层第一部分：LocationManager
这个类用来定位设备位置，需要用到delegate
首先是.h文件：
导入CoreLacation、应用LocationManager类、代理方法申明：
```objc
@import CoreLocation;
@class LocationManager;
@protocol LocationManagerDelegate <NSObject>

@optional

- (void)mapManager:(LocationManager *)manager didUpdateAndGetLastCLLocation:(CLLocation *)location;
- (void)mapManager:(LocationManager *)manager didFailed:(NSError *)error;
- (void)mapManagerServerClosed:(LocationManager *)manager;
- 
@end

@interface LocationManager : NSObject

@property(weak,nonatomic) id <LocationManagerDelegate> delegate;
@property (nonatomic, readonly) CLAuthorizationStatus          authorizationStatus;

- (void)start;
@end

```
.m文件
代理方法
```objc

#pragma mark - 代理方法
- (void)locationManager:(CLLocationManager *)manager didUpdateLocations:(NSArray *)locations
{
    [manager stopUpdatingLocation];
    
    if (_delegate &&[_delegate respondsToSelector:@selector(mapManager:didUpdateAndGetLastCLLocation:)]) {
        CLLocation *location = [locations lastObject];
        [_delegate mapManager:self didUpdateAndGetLastCLLocation:location];
    }
}

- (void)locationManager:(CLLocationManager *)manager didFailWithError:(NSError *)error
{
    NSLog(@"定位失败");
    
    if ([CLLocationManager locationServicesEnabled] == NO) {
        NSLog(@"定位功能关闭");
        if (_delegate && [_delegate respondsToSelector:@selector(mapManagerServerClosed:)]) {
      
            [_delegate mapManagerServerClosed:self];
        }
        
    } else {
        
        NSLog(@"定位功能开启");
        if (_delegate && [_delegate respondsToSelector:@selector(mapManager:didFailed:)]) {
               NSLog(@"%@", error);
            [_delegate mapManager:self didFailed:error];
        }
    }

}

```
数据层第二部分：获取Bing搜索首页的每日一图
首先直接在浏览器访问http://cn.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1，可以看到返回的数据
```js
{"images":[{"startdate":"20160425","fullstartdate":"201604251600","enddate":"20160426","url":"http://s.cn.bing.net/az/hprichbg/rb/Plumeria_ZH-CN10955138144_1920x1080.jpg","urlbase":"/az/hprichbg/rb/Plumeria_ZH-CN10955138144","copyright":"夏威夷鸡蛋花 (© Darrell Gulin/Getty Images)","copyrightlink":"http://www.bing.com/search?q=%E9%B8%A1%E8%9B%8B%E8%8A%B1&form=hpcapt&mkt=zh-cn","wp":true,"hsh":"c73ab3e6b4ef1e09c30e25b407d267e7","drk":1,"top":1,"bot":1,"hs":[],"msg":[{"title":"今日图片故事","link":"http://www.bing.com/search?q=%E9%B8%A1%E8%9B%8B%E8%8A%B1&form=pgbar1&mkt=zh-cn","text":"蛋黄花"}]}],"tooltips":{"loading":"正在加载...","previous":"上一页","next":"下一页","walle":"此图片不能下载用作壁纸。","walls":"下载今日美图。仅限用作桌面壁纸。"}}

```

参数format指的是返回格式 js：jason ； idx：几天前 这里需要获取的是当日图片 故=0；n=1，图片数量。

这里直接取出每日一图的url字段，展示在view上 
```objc
 NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:@"http://cn.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1"]];
    NSData *reponse = [NSURLConnection sendSynchronousRequest:request returningResponse:nil error:nil];
    NSDictionary *picDic = [NSJSONSerialization JSONObjectWithData:reponse options:NSJSONReadingMutableContainers error:&err];
    
    NSDictionary *picc = [picDic objectForKey:@"images"];
    NSArray *arr = picc;
    NSArray *arr1 = arr[0];
    NSDictionary *dic = arr1;
    NSString *picStr = [dic objectForKey:@"url"];
    NSLog(@"picstr----%@",picStr);
    NSURL *picURL = [NSURL URLWithString:picStr];
    UIImageView *picView = [[UIImageView alloc]initWithFrame:self.view.bounds];
    picView.backgroundColor = [[UIColor redColor]colorWithAlphaComponent:0.1];
    picView.contentMode = UIViewContentModeScaleAspectFill;
    [picView sd_setImageWithURL:picURL];
    [self.view addSubview:picView]; 
```


数据层第三部分：GetWeatherData
这个类用get请求获取天气信息，获取到具体城市的天气信息之后，将信息传给其他的model来处理，需要用到delegate
.h文件
代理方法
```objc

@protocol GetWeatherDataDelegate <NSObject>

- (void)sucess:(BOOL)sucess;

@end

```
数据请求方法
```objc
-(void)request: (NSString*)httpUrl withHttpArg: (NSString*)HttpArg ;
- (void)getData;

```
.m文件实现部分
```objc
-(void)request: (NSString*)httpUrl withHttpArg: (NSString*)HttpArg  {
    NSString *urlStr = [[NSString alloc]initWithFormat: @"%@?%@", httpUrl, HttpArg];
    NSURL *url = [NSURL URLWithString: urlStr];
    NSMutableURLRequest *request = [[NSMutableURLRequest alloc]initWithURL: url cachePolicy: NSURLRequestUseProtocolCachePolicy timeoutInterval: 10];
    [request setHTTPMethod: @"GET"];
    [request addValue: APIkey forHTTPHeaderField: @"apikey"];
    [NSURLConnection sendAsynchronousRequest: request
                                       queue: [NSOperationQueue mainQueue]
                           completionHandler: ^(NSURLResponse *response, NSData *data, NSError *error){
                               if (error) {
                                   NSLog(@"Httperror: %@%ld", error.localizedDescription, error.code);
                                   [_delegate sucess:NO];
                               } else {
                                   NSInteger responseCode = [(NSHTTPURLResponse *)response statusCode];
                                   NSError *err;
                                   NSDictionary *dic = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingMutableContainers error:&err];
                                   NSArray *arr = [dic objectForKey:HeWeather];
                                   
                                   NSDictionary *dic0 = arr[0];
  
                                   NSDictionary *dic2 = [dic0 objectForKey:daily_forecast];
//                                   NSArray *arr1 = dic2;
                                   self.weather = dic0;
                                   
                                   self.weatherData = [[WeatherData alloc]initWithDictionary:self.weather];
                                   [_delegate sucess:YES];
                               }
                           }];
}
- (void)getData
{
    NSString *httpUrl = @"http://apis.baidu.com/heweather/weather/free";
//   NSString *httpArg = @"city=hangzhou";
    NSString *httpArg = self.cityStr;
    [self request: httpUrl withHttpArg: httpArg];
//    NSLog(@"test%@",httpArg);
    
    
}
```
这里用到的接口由百度提供，由于该接口请求参数只接受城市的中文名称（比如杭州，而非杭州市）或者拼音，所以需要1.定位到当前地址获得城市名称 2.处理城市名称（杭州市-->杭州） 3.转码之后拼接字符串请求


数据层第四部分：WeatherModels
WeatherModels用来处理天气信息
获得天气数据之后，要用到的主要是两个部分，第一个部分是当前的天气状况，这里需要取值的是气温、湿度、当天的最高低气温、日落日出信息、天气的文字和代码信息、风速。第二个部分是每隔三小时的天气预报（也可以换成可以提供每小时天气预报的API）1.字典转数组 2.分时间 第三个部分是一周天气预报 1.字典转数组 2.分日期（周几）、天气状况、最高最低气温三个部分显示。
接下来依次分析jason返回的数据（可以1.[直接用百度的API调试工具](http://apistore.baidu.com/astore/toolshttpproxy?apiId=usq9zQ&isAworks=1)2.看返回的字典 3.打断点看Auto（更推荐此方法，层次清晰））返回数据太长，不贴出
字典里主要包括七个部分
hourly_forecast：每隔三小时的天气预报（这个要用到，取出来）
status ：状态码 [我用得和风天气API状态码参照](http://apistore.baidu.com/apiworks/servicedetail/478.html)
daily_forecast：一周的天气预报（取出来） 
aqi ：空气质量（这个其实也可以取出来，不过我没用到，暂时不取）
basic ：城市信息，包括经纬度，这里不需要
suggestion ： 生活建议、出行建议、穿衣建议等等
now ：实况天气（取出来）

接下来就是将需要的字典放到各自的类里，也可以只写一个model，我的习惯是分开写，这样出问题的时候方便查找，维护起来简单一些



四、一些思考

1. UI层添加label时有很多重复的代码，可以封装一个继承自UILabel的类，让添加label的工作变得更加简单一点
2.  刷新时界面比较无聊，可以添加一个有意思的等待动画
3.  还可以用tableFooterView添加一些出行建议等实用信息，让这个应用丰满一点

