---
title: 聊天应用ONEChat介绍
date: 2016-03-13 23:41:27
tags:
---
一、数据库与服务器
数据库用的是MySQL 服务器用的是openfire

二、UI搭建
页面仿照微信搭建，第一个页面是好友列表 第二个页面是打算做一个倒计时器，第三个页面是个人信息。
Storyboard部分一：
比较简单，看截图
登录注册Storyboard截图
![](http://ww4.sinaimg.cn/large/0060lm7Tgw1f3bhcm785cj31kw0n7abi.jpg)
实现效果：
![](http://ww1.sinaimg.cn/large/0060lm7Tgw1f3bhdakq2lj30ku112q3u.jpg)
![](http://ww4.sinaimg.cn/large/0060lm7Tgw1f3bhdbn9l8j30ku112q3i.jpg)

Storyboard部分二：
截图
![]()
1.将storyboard原有的ViewController删除，拖入一个TabBarController，接着拖入三个NavigationController，并将底部的item分别命名为好友列表 倒计时 我 ，拖入三个TableView与相应的NavigationController相连，并将NavigationController与TabBarController相连，将TabBarController的“is Initial View Controller”的选项勾选。
2.好友列表中要实现的第一个效果是，当点击某一个好友时，会进入聊天页面，所以拖一个ViewController进去，与好友列表的TableView相连；第二个效果是点击某个图标时，可以添加好友，所以在右上角添加一个button，并拖入一个TableView与之相连。Anyway,所有涉及到TableView的部分，需要设置delegate与datasource。
![](http://ww4.sinaimg.cn/large/0060lm7Tgw1f3bhe8wc01j30ku112gna.jpg)
![](http://ww4.sinaimg.cn/large/0060lm7Tgw1f3bhe42eeoj30ku11275f.jpg)
代码第一部分：
新建类LoginViewController负责登录
新建类RegisterViewController负责注册
代码第二部分：
新建类MeTableViewController负责我的信息
新建类ContactViewController负责好友列表
新建类ChatViewController负责聊天室

三、XMPPTool
逻辑 -> 初始化XMPPStream
 -> 连接到服务器【类似于传一个jid 账号和hostname】
 -> 连接到服务器成功后，再发送密码授权
-> 授权成功后，发送“在线”消息

a 初始化XMPPStream
```objc

-(void)setupXMPPStream
{
    _xmppStream = [[XMPPStream alloc]init];
#warning 每一个模块添加后都要激活
//    添加自动连接模块
    _reconnect = [[XMPPReconnect alloc]init];
    [_reconnect activate:_xmppStream];
    //    添加电子名片模块
    _vCardStorge = [XMPPvCardCoreDataStorage sharedInstance];
    _vCard = [[XMPPvCardTempModule alloc]initWithvCardStorage:_vCardStorge];
    
//    激活
    [_vCard activate:_xmppStream];
    
//    头像模块
    _avatar = [[XMPPvCardAvatarModule alloc]initWithvCardTempModule:_vCard];
    
//    激活
    [_avatar activate:_xmppStream];
    
    
//    添加花名册模块【获取好友列表】
    _rosterStorge = [[XMPPRosterCoreDataStorage alloc]init];
    _roster = [[XMPPRoster alloc]initWithRosterStorage:_rosterStorge];
    [_roster activate:_xmppStream];
    
//    添加聊天模块
    _msgStorge = [[XMPPMessageArchivingCoreDataStorage alloc]init];
    _msgArchiving = [[XMPPMessageArchiving alloc]initWithMessageArchivingStorage:_msgStorge];
    [_msgArchiving activate:_xmppStream];
        
    //    设置代理
    [_xmppStream addDelegate:self delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
}

```
b 连接到服务器
```objc

- (void)connectToHost
{
    NSLog(@"开始连接到服务器");
    if (!_xmppStream) {
        [self setupXMPPStream];
    }
    //    设置JID
    //    resource  标示用户登录的客户端 “iphone”
    
    //    从单例中获取用户名
    
    NSString *user = nil;
    if (self.isRegisterOperation) {
        user = [cCUserInfo sharedcCUserInfo].registerUser;
    }else
    {
        user = [cCUserInfo sharedcCUserInfo].user;
    }
    XMPPJID *myJID = [XMPPJID jidWithUser:user domain:@"127.0.0.1" resource:@"iphone"];
    _xmppStream.myJID = myJID;
    //    设置服务器域名
    _xmppStream.hostName = @"127.0.0.1";//域名或者IP
    
    //设置端口
    _xmppStream.hostPort = 5222;//服务器端口是5222时 可以省略此句
    
    //    连接
    NSError *err = nil;
    if (![_xmppStream connectWithTimeout:XMPPStreamTimeoutNone error:&err]) {
        NSLog(@"%@",err);
    }
}

```

c 连接到服务器成功后，再发送密码授权
```objc
- (void)sendPwdToHost
{
    NSLog(@"发送密码授权");
    NSError *err = nil;
    //    从单例里获取密码
    NSString *pwd = [cCUserInfo sharedcCUserInfo].pwd;
    
    [_xmppStream authenticateWithPassword:pwd error:&err];
    if (err) {
        NSLog(@"%@",err);
    }
}


```

授权成功 授权失败
```objc
- (void)xmppStreamDidAuthenticate:(XMPPStream *)sender
{
    NSLog(@"授权成功");
    [self sendOnlineToHost];
    //回调控制器登录成功
    if (_resultBlock) {
        _resultBlock(XMPPResultTypeLoginSuccess);
    }
}
- (void)xmppStream:(XMPPStream *)sender didNotAuthenticate:(DDXMLElement *)error
{
    //判断block有无值，再回调给登录控制器
    if (_resultBlock) {
        _resultBlock(XMPPResultTypeLoginFailue);
    }
      NSLog(@"授权失败%@",error);
}
```

d 授权成功后 ，发送“在线”消息
```objc
- (void)sendOnlineToHost
{
    NSLog(@"发送 在线 消息");
    XMPPPresence *presence = [XMPPPresence presence];
    NSLog(@"%@",presence);
    [_xmppStream sendElement:presence];
}
```

e 注销方法
```objc
- (void)xmppUserlogout
{
    // 1.发送“离线”消息
    XMPPPresence *offline = [XMPPPresence presenceWithType:@"unavailable"];
    
    [_xmppStream sendElement:offline];
    // 2.与服务器断开连接
    [_xmppStream disconnect];
    //3.回到登录界面
//    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Login" bundle:nil];
//    self.window.rootViewController = storyboard.instantiateInitialViewController;
    [UIStoryboard showInitialVCWithName:@"Login"];
    //    4.更新用户的登录状态
    [cCUserInfo sharedcCUserInfo].loginStatus = NO;
    [[cCUserInfo sharedcCUserInfo]saveUserInfoToSanbox];
}
```

f XMPP请求结果的Block
```objc
typedef enum {
    XMPPResultTypeLoginSuccess,//登录成功
    XMPPResultTypeLoginFailue,//登录失败
    XMPPResultTypeRegisterSuccess,//注册成功
    XMPPResultTypeRegisterFailue,//注册 失败
    XMPPResultTypeNetErr//网络不给力
}XMPPResultType;
```

四、功能实现（before：在服务器注册几个账号测试用）
1. 登录功能
逻辑
（输入账号+密码 -> 判断是否正确-> 是-> 进入 ）

a 新建一个单例UserInfo存放信息
.h
```objc
@property(nonatomic,copy) NSString *user;
@property(nonatomic,copy) NSString *pwd;
/*
 *登录状态
 */
@property (nonatomic,assign) BOOL loginStatus;

```
.m
 ```objc 
 - (void)loadUserInfoFromSandbox
{
    
    NSUserDefaults *defaults= [NSUserDefaults standardUserDefaults];
    self.user = [defaults objectForKey:UserKey];
    self.loginStatus = [defaults boolForKey:LoginStatusKey];
    self.pwd = [defaults objectForKey:pwdKey];
    
}
 ```
b LoginViewController部分
用户名 密码 登录按钮

```objc
@interface ONEChatLoginViewController ()

//@property (weak, nonatomic) IBOutlet UIImageView *avatar;
@property (weak, nonatomic) IBOutlet UITextField *userText;
@property (weak, nonatomic) IBOutlet UITextField *pwdText;
@property (weak, nonatomic) IBOutlet UIButton *loginBtn;

@end
```
 
登陆按钮点击事件

```objc
- (IBAction)loginClick {
    [self.view endEditing:YES];

    
    ONEChatUserInfo *userInfo = [ONEChatUserInfo sharedONEChatUserInfo];
    userInfo.user = self.userText.text;
 
    userInfo.pwd =self.pwdText.text;
   

    [self login];
}

```
登录
```objc
- (void)login
{
    [self.view endEditing:YES];
    __weak typeof(self) selfVc = self;
    
    
    [[ONEChatXMPPTool sharedONEChatXMPPTool] xmppUserLogin:^(XMPPResultType type) {
        [selfVc handleResultType:type];
    }];
    

}

```
登录之后处理
```objc
- (void)handleResultType:(XMPPResultType)type
{
    //主线程刷新UI
    dispatch_async(dispatch_get_main_queue(), ^{
        [MBProgressHUD hideHUDForView:self.view];
        switch (type) {
            case XMPPResultTypeLoginSuccess:
                NSLog(@"登录成功");
                [self enterMainPage];
                break;
            case XMPPResultTypeLoginFailue:
                NSLog(@"登录失败");
                [MBProgressHUD showError:@"用户名或密码不正确"toView:self.view ];
                break;
            case XMPPResultTypeNetErr:
                [MBProgressHUD showError:@"网络炸啦" toView:self.view ];
            default:
                break;
        }
    });
}


```
登录成功
```objc
- (void) enterMainPage{
    //    更改用户的登录状态为yes
    [ONEChatUserInfo sharedONEChatUserInfo].loginStatus = YES;
        //把用户登录成功的数据，保存到沙盒
    [[ONEChatUserInfo sharedONEChatUserInfo]saveUserInfoToSanbox];
    
    //隐藏模态窗口
    [self dismissViewControllerAnimated:NO completion:nil];
    //登录成功来到主界面
    //登录成功后来到主界面
    //    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    //    self.view.window.rootViewController = storyboard.instantiateInitialViewController;
    [UIStoryboard showInitialVCWithName:@"Main"];
}

```

2. 注册功能
逻辑
（输入账号+密码 ->账号是否新用户 +密码是否符合格式-> 是 -> 注册成功）

UserInfo.m
```objc
- (void)saveUserInfoToSanbox
{
    NSUserDefaults *defaults= [NSUserDefaults standardUserDefaults];
    [defaults setObject:self.user forKey:UserKey];
    [defaults setBool:self.loginStatus forKey:LoginStatusKey];
    [defaults setObject:self.pwd forKey:pwdKey];
    [defaults synchronize];
}

```
注册用户名 + 密码 
注册按钮点击
```objc
- (IBAction)registerClick {
    [self.view endEditing:YES];

    ONEChatUserInfo *userInfo = [ONEChatUserInfo sharedONEChatUserInfo];
    userInfo.registerUser = self.registerUser.text;
    userInfo.registerPwd = self.registerPwd.text;
    [ONEChatXMPPTool sharedONEChatXMPPTool].registerOperation = YES;
    __weak typeof(self) selfVc = self;
    [[ONEChatXMPPTool sharedONEChatXMPPTool] xmppUserRegister:^(XMPPResultType type) {
        
        [selfVc handleResultType:type];
    }];    
}

```
注册结果
```objc
- (void)handleResultType:(XMPPResultType)type
{
    
    dispatch_async(dispatch_get_main_queue(), ^{
        
        switch (type) {
            case XMPPResultTypeRegisterSuccess:
                                    [MBProgressHUD showMessage:@"注册成功" toView:self.view];
                NSLog(@"注册成功");
                [self enterMainPage];                break;
                
            case XMPPResultTypeRegisterFailue:
                NSLog(@"注册失败");
                                    [MBProgressHUD showError:@"注册失败"toView:self.view ];
                break;
                
            case XMPPResultTypeNetErr:
                                    [MBProgressHUD showError:@"网络炸啦" toView:self.view ];
            default:
                NSLog(@"break");
                break;
        }
        
    });
}

```
注册成功之后
```objc
- (void) enterMainPage{
    //    更改用户的登录状态为yes
    [ONEChatUserInfo sharedONEChatUserInfo].loginStatus = YES;
    //把用户登录成功的数据，保存到沙盒
    [[ONEChatUserInfo sharedONEChatUserInfo]saveUserInfoToSanbox];
    
    //隐藏模态窗口
    [self dismissViewControllerAnimated:NO completion:nil];
    //登录成功来到主界面
    //登录成功后来到主界面
    //    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    //    self.view.window.rootViewController = storyboard.instantiateInitialViewController;
    [UIStoryboard showInitialVCWithName:@"Main"];
}

```

3. 个人名片 
逻辑
（显示个人信息 + 修改头像 + 修改相应信息）
a 显示相应信息
```objc
- (void)viewDidLoad
{
    [super viewDidLoad];
    self.title = @"我";
    
    XMPPvCardTemp *myVCard = [ONEChatXMPPTool sharedONEChatXMPPTool].vCard.myvCardTemp;
    if (myVCard.photo) {
        self.avatarView.image = [UIImage imageWithData:myVCard.photo];
       
    }
    
    self.nicknameLabel.text = myVCard.nickname;
    NSString *user = [ONEChatUserInfo sharedONEChatUserInfo].user;
    
    self.chatnameLabel.text = [NSString stringWithFormat:@"%@",user];
    
}

```
修改头像图片
点击头像之后弹出ActionSheet
```objc

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
    //获取cell的tag
    UITableViewCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    NSInteger *tag = cell.tag;
    //    判断
    if (tag == 0) {
        //        选择照片
        UIActionSheet *sheet =[[UIActionSheet alloc]initWithTitle:@"请选择" delegate:self cancelButtonTitle:@"取消" destructiveButtonTitle:@"相册" otherButtonTitles:nil, nil];
        [sheet showInView:self.view];
        NSLog(@"这行放图片");
    }
    if (tag == 1){
        //        跳到下一个控制器
        NSLog(@"这行是昵称");
    }else{
        //        什么都不干
        NSLog(@"这行是账号");
    }
}


```
ActionSheet的代理
```objc
- (void)actionSheet:(UIActionSheet *)actionSheet clickedButtonAtIndex:(NSInteger)buttonIndex
{
    UIImagePickerController *imagePicker = [[UIImagePickerController alloc]init];
    //   设置代理
    imagePicker.delegate = self;
    //    设置允许编辑
    imagePicker.allowsEditing = YES;
    
    if (buttonIndex == 0) {
        //相册
        NSLog(@"相册");
        imagePicker.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;
        
    }else{
        //取消
        return;
    }
    //    显示图片选择器
    [self presentViewController:imagePicker animated:YES completion:nil];
}
```
图片选择器的代理
```objc
- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary<NSString *,id> *)info
{
    NSLog(@"%@",info);
    //    获取图片 设置图片
    UIImage *image = info[UIImagePickerControllerEditedImage];
        self.avatarView.image = image;
        //    隐藏窗口
    [self dismissViewControllerAnimated:YES completion:nil];
    XMPPvCardTemp *myvCard = [ONEChatXMPPTool sharedONEChatXMPPTool].vCard.myvCardTemp;
    myvCard.photo =UIImagePNGRepresentation(image);
        [[ONEChatXMPPTool sharedONEChatXMPPTool].vCard updateMyvCardTemp:myvCard];    
}

```
4. 好友列表 
逻辑
（获取好友 并以某种形式排列在view中）
a.使用coredata获取数据
 1.上下文【关联到数据】
 
 ```objc
    NSManagedObjectContext *context = [cCXMPPTool sharedcCXMPPTool].rosterStorge.mainThreadManagedObjectContext;
 ```  
  
 2.fetchrequest
 
 ```objc
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"XMPPUserCoreDataStorageObject"];
 ```   

3.设置过滤和排序

```objc
    //    过滤当前登录用户的好友
    NSString *jid = [cCUserInfo sharedcCUserInfo].jid;
    
    NSPredicate *pre = [NSPredicate predicateWithFormat:@"streamBareJidStr = %@",jid];
    request.predicate = pre;
    //    排序
    NSSortDescriptor *sort = [NSSortDescriptor sortDescriptorWithKey:@"displayName" ascending:YES];
    request.sortDescriptors = @[sort];
 ```   

4.执行请求获取数据

```objc
    _resultsControl = [[NSFetchedResultsController alloc]initWithFetchRequest:request managedObjectContext:context sectionNameKeyPath:nil cacheName:nil];
    _resultsControl.delegate = self;
    NSError *err = nil;
    
    [_resultsControl performFetch:&err];
    if (err) {
        NSLog(@"%@",err);
    }
 ```
 
b.数据内容改变时调用的方法

```objc
- (void)controllerDidChangeContent:(NSFetchedResultsController *)controller
{
    NSLog(@"changeeeee");
//    刷新表格
    [self.tableView reloadData];
}
```

c. 表格显示用户账户名 在线状态

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    static NSString *ID =@"contactcell";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:ID];
    
//    sectionnum
//    0:在线
//    1:离开
//    2:离线
    
//    获取对应好友
    XMPPUserCoreDataStorageObject *friend =_resultsControl.fetchedObjects[indexPath.row];
    
    switch ([friend.sectionNum intValue]) {//好友状态
        case 0:
            cell.detailTextLabel.text = @"在线";
            break;
        case 1:
            cell.detailTextLabel.text = @"离开";
            break;
        case 2:
            cell.detailTextLabel.text = @"离线";
            break;
        default:
            break;
    }
      cell.textLabel.text = friend.jidStr;
     
    NSLog(@"获取好友");
     return cell;
    }
```
    
d 表格行数由好友数量决定

```objc
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    return _resultsControl.fetchedObjects.count;
}

```
e 删除好友

```objc
- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath
{
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        NSLog(@"删除好友");
        
        XMPPUserCoreDataStorageObject *friend = _resultsControl.fetchedObjects[indexPath.row];
        XMPPJID *friendJid = friend.jid;
        [[cCXMPPTool sharedcCXMPPTool].roster removeUser:friendJid];
    }
    
}
```

5. 添加好友 
逻辑
（搜索账号 -> 是否存在用户 -> 是 -> 订阅该用户）

点击添加按钮之后 输入账号

```objc
- (BOOL)textFieldShouldReturn:(UITextField *)textField
{
//添加好友
//1获取好友账号
    NSString *user = textField.text;
    NSLog(@"%@",user);
//    判断是否添加自己
    if ([user isEqualToString:[cCUserInfo sharedcCUserInfo].user]) {
        [self showAlert:@"不能添加自己为好友"];
        return YES;
    }
    
    NSString *jidStr = [NSString stringWithFormat:@"%@@%@",user,domain];
    XMPPJID *friendJid = [XMPPJID jidWithString:jidStr];
//    判断好友是否已经存在
    if ([[cCXMPPTool sharedcCXMPPTool].rosterStorge userExistsWithJID:friendJid xmppStream:[cCXMPPTool sharedcCXMPPTool].xmppStream]) {
        [self showAlert:@"当前好友已经存在"];
        return YES;
    }
    
//2发送添加好友的请求
//    添加好友 xmpp订阅
   
    [[cCXMPPTool sharedcCXMPPTool].roster subscribePresenceToUser:friendJid];
    return YES;
}
```
AlertView

```objc
-(void)showAlert:(NSString *)msg{
    UIAlertView *alert = [[UIAlertView alloc]initWithTitle:@"温馨提示" message:msg delegate:nil cancelButtonTitle:@"谢谢" otherButtonTitles:nil, nil];
    [alert show];
}

```
6. 发送消息之text 
逻辑
（textfield输入文字 -> 发送消息-> 接收消息 -> 显示消息）
a 选中好友进入聊天

```objc
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
//    获取好友
     XMPPUserCoreDataStorageObject *friend =_resultsControl.fetchedObjects[indexPath.row];
//    选中表格进行聊天界面
    [self performSegueWithIdentifier:@"ChatSegue" sender:friend.jid];
}

```
b 加载聊天记录
使用coredata获取数据

```objc
- (void)loadMsgs
{
//上下文
    NSManagedObjectContext *context = [cCXMPPTool sharedcCXMPPTool].msgStorge.mainThreadManagedObjectContext;
//    请求对象
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"XMPPMessageArchiving_Message_CoreDataObject"];
    
//    过滤、排序
//    1.当前登录用户的jid的消息
//    2。好友的jid消息

    NSPredicate *pre = [NSPredicate predicateWithFormat:@"streamBareJidStr = %@ AND bareJidStr = %@",[cCUserInfo sharedcCUserInfo].jid,self.friendJid.bare];
    request.predicate = pre;
    
//时间升序
    NSSortDescriptor *timeSort = [NSSortDescriptor sortDescriptorWithKey:@"timestamp" ascending:YES];
    request.sortDescriptors = @[timeSort];
    
//    查询
    _resultContr = [[NSFetchedResultsController alloc]initWithFetchRequest:request managedObjectContext:context sectionNameKeyPath:nil cacheName:nil];
    NSError *err = nil;
    [_resultContr performFetch:&err];
    _resultContr.delegate = self;
    if (err) {
        NSLog(@"%@",err);
    }
}

```
表格的数据源
```objc
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    return _resultContr.fetchedObjects.count;
}
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    static NSString *ID =@"ChatCell";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:ID];
    if (!cell) {
        cell = [[UITableViewCell alloc]initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:ID];
    }
//    获取聊天消息对象
    XMPPMessageArchiving_Message_CoreDataObject *msg = _resultContr.fetchedObjects[indexPath.row];
    
//    判断是图片还是纯文本
    NSString *chatType = [msg.message attributeStringValueForName:@"chatType"];
    if ([chatType isEqualToString:@"image"]) {
//    TODO    图片显示
      
    }
    else
//       else if ([chatType isEqualToString:@"text"])
        {
        
        //    显示
        if ([msg.outgoing boolValue]) {//自己发
            cell.textLabel.text = msg.body;
            
//            cell.textLabel.text = [NSString stringWithFormat:@"me: %@",msg.body];
        }else
        {
            //        别人发
            cell.textLabel.text = msg.body;

//            cell.textLabel.text = [NSString stringWithFormat:@"other: %@",msg.body];
        }
        cell.imageView.image = nil;
           }
       return cell;
  }

```
如果消息有更新，则scrollerView滚动到底部


```objc
- (void)scrollToTableBottom
{
        NSInteger lastRow = _resultContr.fetchedObjects.count - 1;
    if (lastRow < 0 ) {
        return;
    }
    NSIndexPath *lastPath = [NSIndexPath indexPathForRow:lastRow inSection:0];
    [self.tableView scrollToRowAtIndexPath:lastPath atScrollPosition:(UITableViewScrollPositionBottom) animated:YES];
}
```
ResultsController代理

```objc

- (void)controllerDidChangeContent:(NSFetchedResultsController *)controller
{
    [self.tableView reloadData];
    [self scrollToTableBottom];
}

```
c textfield代理

```objc
- (void)textViewDidChange:(UITextView *)textView
{
//    获取contentsize
    CGFloat contentH = textView.contentSize.height;
    NSLog(@"%f",contentH);
//    NSLog(@"%@",textView.text);
    
//    大于33：超过一行的高度 小于68：三行内
    if (contentH > 33 && contentH < 68) {
        self.inputViewHeightConstraint.constant = contentH + 18;
    }
    NSString *text = textView.text;
//    换行等于send
    if ([text rangeOfString:@"\n"].length != 0) {
        NSLog(@"send msgs %@",text);
        
//        去除换行字符
        text = [text stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
        
        [self sengMsgsWithText:text bodyType:@"text"];
//        清空数据
        textView.text = nil;
//        发送完消息 把inputview的高度改回来
        self.inputViewHeightConstraint.constant = 50;
    }else{
        NSLog(@"%@",textView.text);
    }
}
```
d 发送聊天消息

```objc
- (void)sengMsgsWithText:(NSString *)text bodyType:(NSString *)bodyType
{
    XMPPMessage *msg = [XMPPMessage messageWithType:@"chat" to:self.friendJid];
//    text :纯文本
//    image 图片

    [msg addAttributeWithName:@"bodyType" stringValue:bodyType];
//    设置内容
    [msg addBody:text];
    NSLog(@"%@",msg);
    [[cCXMPPTool sharedcCXMPPTool].xmppStream sendElement:msg];
}
```

7. 发送消息之语音 
逻辑
（判断消息类型+发送方 -> 获取消息-> 时间排序 -> 以正确颜色、形式、位置显示在cell中）






8. 发送消息之图片

逻辑
(选择图片 -> 转化成base64编码 -> 设置节点内容 -> 包含子节点 -> 发送消息 -> 接收消息 -> 取出消息解码 -> 图片在label中显示)
点击选择图片按钮
图片代理
```objc

- (void) imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary<NSString *,id> *)info
{
    UIImage *image = info[UIImagePickerControllerOriginalImage];
    NSData *data = UIImagePNGRepresentation(image);
    
    [self sendMessageWithData:data bodyName:@"image"];
    
    [self dismissViewControllerAnimated:YES completion:nil];
}

```
 
 发送二进制文件

```objc
- (void)sendMessageWithData:(NSData *)data bodyName:(NSString *)name
{
     XMPPMessage *message = [XMPPMessage messageWithType:@"chat" to:self.friendjid];
    
    [message addBody:name];
    
    // 转换成base64的编码
    NSString *base64str = [data base64EncodedStringWithOptions:0];
    
    // 设置节点内容
    XMPPElement *attachment = [XMPPElement elementWithName:@"attachment" stringValue:base64str];
    
    // 包含子节点
    [message addChild:attachment];
    
    // 发送消息
    [[ONEChatXMPPTool sharedONEChatXMPPTool].xmppStream sendElement:message];
}
```
TODO 图片显示
```objc
 // 取出消息的解码
                NSString *base64str = node.stringValue;
                NSData *data = [[NSData alloc]initWithBase64EncodedString:base64str options:0];
                UIImage *image = [[UIImage alloc]initWithData:data];
                
                // 把图片在label中显示
                NSTextAttachment *attach = [[NSTextAttachment alloc]init];
                attach.image = [image scaleToSize:CGSizeMake(252.0f, 192.0f)];
//                attach.image = image;
                
                NSAttributedString *attachStr = [NSAttributedString attributedStringWithAttachment:attach];
                cell.textLabel.attributedText = attachStr;
                
                [self.view endEditing:YES];
```

五、改善


 


