---
layout: post
title: Linphone SDK Swift 项目移植
subtitle:   "将 linphone sdk 移植到自己的 swift 项目，使用一种更好的封装方法"
date:       2016-08-10
author:     "Robert Zhang"
header-img: "img/contact-bg.jpg"
tags:
    - IOS
---

> __背景__：需要将 linphone 语音相关功能移植到已有项目中，为了尽可能少移植无用代码，故考虑只移植 linphone sdk 和其必要的调用逻辑代码。  
> __思路__：已有项目为 swift 开，首选将 linphone 移植内容封装成静态库，供 swift 调用。

正式开始前要先吐槽一下 linphone 太过复杂，不经意间就会被绕进去，然后就再也出不来了。还好我只是浅尝辄止，并未过深研究。也许深入后会有新的一片天地，那就看以后有没有相关需要吧！

## 文献

一般大家写博文习惯将鸣谢和文献写在最后。一是为了不破坏文章整体结构的流畅，二是为了对参考文献原作者的尊重。而这篇博文则将文献放在了开始的地方。这是因为，我不想赘述 linphone 是什么，如何使用，如何移植。需要的话通过下面的链接，你可以获得更多的帮助和启示。

linphone 相关博文：

* 官网：[linphone.org](linphone.org)
* linphone SDK 官方下载：[snapshots-ios](https://www.linphone.org/snapshots/ios/)
* [Linphone-iOS-移植](http://blog.csdn.net/frf881128/article/details/50234479)
* [怎样将Linphone移植到自己的项目](http://www.jianshu.com/p/845cd812fcd7) 值得推荐

## 我的封装

### linphone SDK 封装静态库

通过 xcode 的 File -> New -> Project -> 选择 IOS Framework & Library 下的 Cocoa Touch Static Library 创建一个新的项目。当然这个项目可以是独立的，也可以和你原有项目在同一个 workspace，我的项目是后者。
linphone 静态库工程结构如下图：
![]({{ site.baseurl }}/img/2016-8-10-Linphone-SDK-Swift-项目移植.jpeg
)
这里没什么好说的，基本结构和方法与参考博文 [怎样将Linphone移植到自己的项目](http://www.jianshu.com/p/845cd812fcd7) 中的开源项目基本一致。但由于我的已有项目是 Swift 开发的，并不想使用除 linphone sdk 必须的调用文件以外的其他文件。所以并没有使用 ColorSpaceUtilites，UCSIPCCSDKLog 和 UCSIPCCDelegate(我使用系统通知进行状态更新)。
另外代码中不同的地方有：

~~~ 
// LinphoneSDK 等价于 UCSIPCCManager
// LinphoneSDK.h 添加了micro 和 pauseCall 的操作
@property (nonatomic, assign) BOOL microEnabled;                            // 是否打开扬声器
@property (nonatomic, assign) BOOL pauseCallEnabled;                        // 是否打开暂停开关

// LinphoneSDK.m
/**
 @author robertzhang, 16-08-02
 
 麦克风状态
 */
- (BOOL)microEnabled {
    return linphone_core_is_mic_muted([LinphoneManager getLc]) == false;
}
- (void)setMicroEnabled:(BOOL)microEnabled {
    linphone_core_mute_mic([LinphoneManager getLc], microEnabled);
}

/**
 @author robertzhang, 16-08-02
 
 通话暂停键状态
 */
- (BOOL)pauseCallEnabled {
    bool ret = false;
    LinphoneCall *c = [self getCall];
    if (c != nil) {
        LinphoneCallState state = linphone_call_get_state(c);
        ret = (state == LinphoneCallPaused || state == LinphoneCallPausing);
    }
    return ret;
}
- (void)setPauseCallEnabled:(BOOL)pauseCallEnabled {
    LinphoneCall *currentCall = [self getCall];
    if (currentCall != nil) {
        if (pauseCallEnabled) {
            linphone_core_pause_call(LC, currentCall); // 暂停
        } else {
            linphone_core_resume_call(LC, currentCall); // 取消暂停
        }
    }
}
- (LinphoneCall *)getCall {
    LinphoneCall *currentCall = linphone_core_get_current_call(LC);
    if (currentCall == nil && linphone_core_get_calls_nb(LC) == 1) {
        currentCall = (LinphoneCall *)linphone_core_get_calls(LC)->data;
    }
    return currentCall;
}

~~~

另外，因为静态库无法打包进资源文件，所以音频文件需要通过bundle进行管理和使用。LinphoneManager 也需要做相应的修改

~~~
// LinphoneManager.m 加载音频的地方修改如下
    NSBundle *libs = [NSBundle bundleWithPath:[[NSBundle mainBundle] pathForResource:@"Resources" ofType:@"bundle"]];
    
    const char* lRing = [[libs pathForResource:@"ring" ofType:@"wav"] cStringUsingEncoding:[NSString defaultCStringEncoding]];
	linphone_core_set_ring(theLinphoneCore, lRing);

    const char* lRingBack = [[libs pathForResource:@"ringback" ofType:@"wav"] cStringUsingEncoding:[NSString defaultCStringEncoding]];
	linphone_core_set_ringback(theLinphoneCore, lRingBack);

    const char* lPlay = [[libs pathForResource:@"hold" ofType:@"wav"] cStringUsingEncoding:[NSString defaultCStringEncoding]];
	linphone_core_set_play_file(theLinphoneCore, lPlay);
~~~
静态库的创建需要十分小心配置文件的设置，很容易因为被坑到。我就是因为没有在 Compile Sources 中将新添加的文件引入，导致报了很多莫名的错误。当然这和我自己粗心有很大关系。

### Swift 使用静态库

静态库引入已有工程中如果出现 ，xcode 提示 linphoneSDK.a 文件不存在等这类问题，那是因为对于新 Clone 的代码，linphoneSDK.a 的引用失效所致。如果 linphoneSDK 工程和已有工程在同一个 workspace，只需要先编译一下 linphoneSDK 工程，然后将生成的 linphoneSDK.a 文件以及必要的 include 文件重新引入即可。具体的步骤可参看：[JZPhone 的 README](https://github.com/chenzht/JZPhone)

以上是我在 Swift 项目中对 linphoneSDK 的一层封装

~~~
/*
 这里用 Delegate 是为了 VOIP 接口的易扩展，VoipDelegate 相当于一个适配器。
 当需要接入新的 VOIP 组件，只需要创建一个接口类，并实现 VoipDelegate 协议。
 eg: 创建 linphoneSDK.swift ，继承VoipDelegate。通过实现协议中的方法，调用linphone SDK动态库中的 OC 代码完成功能。
 */
/// Voip 组件协议
protocol VoipDelegate {
    /// 初始化 voip 组件
    func initVOIP()
    /// 登陆
    func login(name: String, password: String)
    /// 登出
    func logout()
    /// 拨号
    func onCall(address: String, displayName: String? , transfer: Bool?)
    /// 接电话
    func acceptCall()
    /// 挂断
    func hangUpCall()
    /// 获取通话时长
    func getCallDuration() -> Int
    /// 获取对方号码
    func getRemoteAddress() -> String?
    /// 获取对方昵称
    func getRemoteDisplayName() -> String?
    /// 打开、关闭扬声器
    func switchSpeaker()
    /// 查看扬声器状态
    func isSpeakerOpened() -> Bool
    /// 打开、关闭麦克风
    func switchMicro()
    /// 查看麦克风状态
    func isMicroOpened() -> Bool
    /// 通话挂起
    func switchPauseCall()
    /// 通话挂起状态
    func isPauseCall() -> Bool
}

// CallHelper.swift
class CallHelper {
 ...
 
 /// 通过协议调用相关操作
    static var voipDelegate: VoipDelegate?
 
 ...
}
~~~

实现 VoipDelegate 协议接入 linphoneSDK

~~~
class LinphoneHelper: VoipDelegate {
    
    let domain = "xxx.xxx.xxx.xx"
    let port = "xxxx"
    let transport = "UDP"
    
    
    class var sharedInstance: LinphoneHelper {
        struct Static {
            static let instance: LinphoneHelper = LinphoneHelper()
        }
        return Static.instance
    }
    
// MARK: - Implements VoipDelegate
    /// 初始化 voip 组件
    func initVOIP() {
        LinphoneSDK.instance().startLPhone()
    }
    /// 登陆
    func login(name: String, password: String) {
        LinphoneSDK.instance().addProxyConfig(name, password: password, displayName: name, domain: domain, port: port, withTransport: transport)
    }
    /// 登出
    func logout() {
        LinphoneSDK.instance().removeAccount()
    }
    /// 拨号
    func onCall(address: String, displayName: String? , transfer: Bool?) {
        LinphoneSDK.instance().call(address, displayName: address, transfer: false)
    }
    /// 接电话
    func acceptCall() {}
    func acceptCall(notif: NSNotification) {
        LinphoneSDK.instance().acceptCall(notif)
    }
    /// 挂断
    func hangUpCall() {
        LinphoneSDK.instance().hangUpCall()
    }
    /// 获取通话时长
    func getCallDuration() -> Int {
        let duration = Int(LinphoneSDK.instance().getCallDuration())
        return duration
    }
    /// 获取对方号码
    func getRemoteAddress() -> String? {
       let address = LinphoneSDK.instance().getRemoteAddress()
        return  address
    }
    /// 获取对方昵称
    func getRemoteDisplayName() -> String? {
        let displayname = LinphoneSDK.instance().getRemoteDisplayName()
        return displayname
    }
    /// 打开、关闭扬声器
    func switchSpeaker() {
        LinphoneSDK.instance().speakerEnabled = LinphoneSDK.instance().speakerEnabled ? false : true
    }
    /// 查看扬声器状态
    func isSpeakerOpened() -> Bool {
        return LinphoneSDK.instance().speakerEnabled
    }
    /// 打开、关闭麦克风
    func switchMicro() {
        LinphoneSDK.instance().microEnabled = LinphoneSDK.instance().microEnabled ? true : false
    }
    /// 查看麦克风状态
    func isMicroOpened() -> Bool {
        return LinphoneSDK.instance().microEnabled
    }
    /// 通话挂起
    func switchPauseCall() {
        LinphoneSDK.instance().pauseCallEnabled = LinphoneSDK.instance().pauseCallEnabled ? false : true
    }
    /// 通话挂起状态
    func isPauseCall() -> Bool {
        return LinphoneSDK.instance().pauseCallEnabled
    }
    /// 环境是否准备成功
    func isSDKReady() -> Bool {
        return LinphoneSDK.instance().isLPReady
    }
    /// 获取用户名或地址（sip账号）
    func getName() -> String{
        if let name = LinphoneHelper.sharedInstance.getRemoteDisplayName() where (name as NSString).length > 0 {
           return name
        } else {
            let address = LinphoneHelper.sharedInstance.getRemoteAddress()! as NSString
            return address.substringToIndex(address.rangeOfString("@").location)
        }
    }
    
}
~~~

linphoneSDK 静态库是使用 OC 编写的，Swift 要调用就必须加入桥接，具体办法请自行查阅在此不一赘述。到这里就完成了封装工作。按照我个人的习惯，为了不破坏已有代码的逻辑，也为了让两个关系不大的模块尽可能的低耦合，做一层层的封装还是有必要的。
细心的你一定在上面发现了 

~~~
static var voipDelegate: VoipDelegate? 
~~~

这段代码她是做什么用的呢？没错，她就是我们解耦的关键。

~~~
static var voipDelegate: VoipDelegate? = LinphoneHelper.sharedInstance
~~~

我们只需要替换实现 voipDelegate 协议的的实例对象，就可以自如的切换不同的 VOIP 组件，是不是很神奇呢？剩下的就不用我多说了，在你需要的地方尽情使用 voipDelegate 来调用 VOIP 的功能吧。

## 说明

以上开发环境如下：

>swift: 2.2  
>xcode: 7.3  
>linphone 版本: 老的版本  

替换到最新版本 linphone sdk 需要相应的替换 linphoneManager 文件。


