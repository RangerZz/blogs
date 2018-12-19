---
date: 2018-10-23
title: "ReplayKit 2录屏直播"
tags:
    - ReplayKit
    - 录屏直播
categories:
    - 直播
comment: true
---
## 概览
iOS 11新添加了用户从控制中心开启全局的录屏功能(在此之前只能实现应用内录制),这个功能和API(ReplayKit 2)的添加,使得录屏直播以及其他录屏相关的玩法拥有了无限可能.
刚好近一个月我司产品添加相关功能,并由我完成,从查阅相关文档到添加录屏直播的第一版App上线,用了2周多时间,期间尝试了阿里和腾讯的SDK,下面就回顾一下这段时间的学习和成长.

## 介绍Extension
Xcode 9中,添加了'Broadcast Upload Extension' 和 'Broadcast Setup UI Extension',用来实现相关功能,需要在File -> New -> Target中创建他们.
### Broadcast Upload Extension
最为核心的是'Broadcast Upload Extension'(后简称'Upload'),'Upload'中有多个系统回调方法,被用来告知录屏的开始,结束,暂停,恢复和采集到的音视频信息,大多数需求要通过该Extension实现.
'Upload'独立于App存在,运行时也不依赖App,是一个独立的进程,也就是说,用户并不需要启动App,就可以使用录屏功能,拥有便捷,轻量的特点.
### Broadcast Setup UI Extension
在创建'Broadcast Upload Extension'时,Xcode会询问你是否需要一并创建'Broadcast Setup UI Extension'(后称'Setup UI'),此处可以选择不创建.
'Setup UI'在iOS11添加ReplayKit 2之前需要使用,在ReplayKit 2后,用户可以在控制中心发起全局的录屏功能后本插件就没那么重要了,由于本文主要介绍ReplayKit 2,故不做详细介绍了(我觉得这是个遗留问题,ReplayKit 2新特性更好更方便,完全可以替代之前的方案).

## 实践实现
### 类和方法
创建'Upload'Target后,Xcode中会创建SampleHandler类,是RPBroadcastSampleHandler的子类,类中会有如下方法

```
- (void)broadcastStartedWithSetupInfo:(NSDictionary<NSString *,NSObject *> *)setupInfo {
    // User has requested to start the broadcast. Setup info from the UI extension can be supplied but optional.
}

- (void)broadcastPaused {
    // User has requested to pause the broadcast. Samples will stop being delivered.
}

- (void)broadcastResumed {
    // User has requested to resume the broadcast. Samples delivery will resume.
}

- (void)broadcastFinished {
    // User has requested to finish the broadcast.
}

- (void)processSampleBuffer:(CMSampleBufferRef)sampleBuffer withType:(RPSampleBufferType)sampleBufferType {
    
    switch (sampleBufferType) {
        case RPSampleBufferTypeVideo:
            // Handle video sample buffer
            break;
        case RPSampleBufferTypeAudioApp:
            // Handle audio sample buffer for app audio
            break;
        case RPSampleBufferTypeAudioMic:
            // Handle audio sample buffer for mic audio
            break;
            
        default:
            break;
    }
}

```
根据方法名和注释,显而易见的知道这些方法分别在录屏开始,暂停,恢复,结束和音视频数据是被调用,并且会传递一些信息.
简单说一下这几个方法:

* `-(void)broadcastStartedWithSetupInfo:(NSDictionary<NSString *,NSObject *> *)setupInfo;`本方法在用户在 控制中心->屏幕录制->开始直播 点击321倒数结束后被调用.此处就可以开始初始化推流器,开启推流等工作,至于setupInfo,是'Setup UI'传递过来的一些初始化信息,当然如果不使用'Setup UI',还是有多种方式传递'Upload'插件本身无法获取的信息.
    
*  `-(void)broadcastFinished;`本方法在用户 点击屏幕状态栏->系统弹窗->停止 后被调用.此处可以认为'Upload'插件进程即将被结束,需要做一些资源和内存释放等工作,然后插件进程结束.
    
* `-(void)processSampleBuffer:(CMSampleBufferRef)sampleBuffer withType:(RPSampleBufferType)sampleBufferType;`本方法在`-(void)broadcastStartedWithSetupInfo:(NSDictionary<NSString *,NSObject *> *)setupInfo;`被调用后,会不停的被调用,用来获取采集到的音视频信息,通过sampleBufferType区分.此处可以将音视频信息编码推流.

### 消息传递与数据共享
前文说的,'Upload'和App是相互独立的两个进程,'Upload'和App如何进行消息传递和数据共享呢,大致有如下几种方式:

* 剪贴版:通过写入剪贴板,然后插件读取剪贴板传递一些简单数据,但是可靠性太差,可以作为一种辅助手段.

```
UIPasteboard *paste = [UIPasteboard generalPasteboard];
paste.string = @"a";// 写入
NSString *a = paste.string;// 读取
```

* 本地推送:插件可以发送本地通知,可以用来提示用户一些录屏事件,也可以提示用户点击通知,激活App做一些操作.

```
- (void)sendLocalNotificationToHostAppWithTitle:(NSString *)title msg:(NSString *)msg userInfo:(NSDictionary *)userInfo {
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = [NSString localizedUserNotificationStringForKey:title arguments:nil];
    content.body = [NSString localizedUserNotificationStringForKey:msg  arguments:nil];
    content.sound = [UNNotificationSound defaultSound];
    content.userInfo = userInfo;
    UNTimeIntervalNotificationTrigger* trigger = [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:0.1f repeats:NO];
    UNNotificationRequest* request = [UNNotificationRequest requestWithIdentifier:@"ReplayKitPush" content:content trigger:trigger];   
    [center addNotificationRequest:request withCompletionHandler:^(NSError * _Nullable error) { 
    }];
}
```
* 通知:CFNotificationCenter可以发送跨进程的通知,用来做一些实时事件交互.

```
CFNotificationCenterPostNotification(CFNotificationCenterGetDarwinNotifyCenter(),kDarvinNotificationNamePushStart,NULL,nil,YES);
CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), (__bridge const void *)(self), onDarwinReplayKitPushStart, kDarvinNotificationNamePushStart, NULL, CFNotificationSuspensionBehaviorDeliverImmediately);
```
* AppGroup:开启AppGroup后,同一AppGroup的成员,可以共享沙盒数据,也就是说,App可以读取"Upload"的录屏文件,"Upload"可以从App获取必要信息.AppGroup需要在 Capabilities -> AppGroup 中开启,App和插件都要打开,后会生成.entitlements文件,会有一个group.BundleIdentifier的标识.

```
NSURL *containerURL = [[NSFileManager defaultManager] containerURLForSecurityApplicationGroupIdentifier:AppGroupId];// 共享的沙盒路径
NSUserDefaults *userDefault = [[NSUserDefaults alloc] initWithSuiteName:AppGroupId];// 共享的userDefault
```

### 注意线程问题
"Upload"中开始各种方法被调用的线程都不是主线程,所以要注意线程同步问题,并且实测如果强行回调到主线做一些耗时操作,会使"Upload"崩溃,所以非不得已("Upload"也没有需要处理的UI事件),尽量使用当前线程,并且在开始和结束的方法中加同步锁.
以及如果开启NSTimer的话,由于不是主线程,记得开启当前线程的Runloop!
```
- (void)setupTimerThread {
    if (self.tiemrThread.executing) {
        return;
    }
    
    self.tiemrThread = [[NSThread alloc] initWithTarget:self selector:@selector(startTimer) object:nil];
    self.tiemrThread.name = @"timerThread";
    [self.tiemrThread start];
}

- (void)startTimer {
    @autoreleasepool {

        [self postPushStatus:ANPushHeartbeatStatusPushing];
        self.timer = [NSTimer timerWithTimeInterval:15 repeats:YES block:^(NSTimer * _Nonnull timer) {
        }];
    
        [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSDefaultRunLoopMode];

        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    }
}

- (void)stopTimer {
    [self performSelector:@selector(invalidateTimer) onThread:self.tiemrThread withObject:nil waitUntilDone:NO];
}

- (void)invalidateTimer {
    if (self.timer) {
        [self.timer invalidate];
        self.timer = nil;
    }
}
```

以上是一些开发过程中学习到知识与坑点,如有问题,欢迎交流探讨.
## 参考链接
[Live Screen Broadcast with ReplayKit](https://developer.apple.com/videos/play/wwdc2018/601/)

[What's New with Screen Recording and Live Broadcast](https://developer.apple.com/videos/play/wwdc2017/606)

[腾讯云ReplayKit文档](https://cloud.tencent.com/document/product/454/7883)

[阿里云推流SDK文档](https://help.aliyun.com/document_detail/45263.html?spm=a2c4g.11186623.6.808.1b50592cSXAs1a)
