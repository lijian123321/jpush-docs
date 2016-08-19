# iOS SDK 集成指南

<style>
img[alt=jpush_ios] { width: 800px; }
</style>

## SDK说明

### 适用版本

本文匹配的 SDK版本：r2.1.5 以后。  
[查看最近更新](http://docs.jiguang.io/updates/)了解最新的SDK更新情况。  
使用Xcode 6及以上版本可以使用新版Push SDK，Xcode 5环境下需要运行旧版本SDK(1.7.4)


### 资源文件

包名为JPush-iOS-SDK-{版本号}

* lib文件夹：包含头文件 JPUSHService.h，静态库文件jpush-ios-x.x.x.a ，支持的iOS版本为 6.0 及以上版本。（请注意：模拟器不支持APNs）
* pdf文件：集成指南
* demo文件夹：示例

## 创建应用
* 在 JPush的管理Portal 上创建应用并上传APNs证书。如果对APNs证书不太了解 请参考： [iOS 证书设置指南](ios_cer_guide) 

![jpush_ios][0]

* 创建成功后自动生成 AppKey 用以标识该应用。 

![jpush_ios][1]

## 配置工程
### 导入SDK

* 将SDK包解压，在Xcode中选择“Add files to 'Your project name'...”，将解压后的lib子文件夹（包含JPUSHService.h、jpush-ios-x.x.x.a）添加到你的工程目录中。  
* 添加FrameWork
  * CFNetwork.framework
  * CoreFoundation.framework
  * CoreTelephony.framework
  * SystemConfiguration.framework
  * CoreGraphics.framework
  * Foundation.framework
  * UIKit.framework
  * Security.framework
  * Xcode7需要的是libz.tbd；Xcode7以下版本是libz.dylib
  * Adsupport.framework (获取IDFA需要；如果不使用IDFA，请不要添加)
  * UserNotifications.framework(Xcode8及以上)

### Build Settings
如果你的工程需要支持小于7.0的iOS系统，请到Build Settings 关闭 bitCode 选项，否则将无法正常编译通过。

* 设置 Search Paths 下的 User Header Search Paths 和 Library Search Paths，比如SDK文件夹（默认为lib）与工程文件在同一级目录下，则都设置为"$(SRCROOT)/{静态库所在文件夹名称}"即可。

### Capabilities
如使用Xcode8及以上环境开发，请开启Application Target的Capabilities->Push Notifications选项，如图：  
![jpush_ios][7]

### 允许Xcode7支持Http传输方法

如果用的是Xcode7时，需要在App项目的plist手动配置下key和值以支持http传输:

**选择1：根据域名配置**

* 在项目的info.plist中添加一个Key：NSAppTransportSecurity，类型为字典类型。
* 然后给它添加一个NSExceptionDomains，类型为字典类型；
* 把需要的支持的域添加給NSExceptionDomains。其中jpush.cn作为Key，类型为字典类型。
* 每个域下面需要设置2个属性：NSIncludesSubdomains、NSExceptionAllowsInsecureHTTPLoads。
两个属性均为Boolean类型，值分别为YES、YES。

如图：

![jpush_ios][6]


**选择2：全局配置**

```
  <key>NSAppTransportSecurity</key> 
  <dict> 
  	<key>NSAllowsArbitraryLoads</key> 
  	<true/> 
  </dict>            
```


## 添加代码

<div style="font-size:13px;background: #E0EFFE;border: 1px solid #ACBFD7;border-radius: 3px;padding: 8px 16px; padding-bottom: 0;margin-bottom: 0;">	<p>2.1.0版本开始,API类名为JPUSHService，不再使用原先的APService。
</div>

### 初始化代码

请将以下代码添加到  
-(BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions

```   	

  NSString *advertisingId = [[[ASIdentifierManager sharedManager] advertisingIdentifier] UUIDString];
  //Required
  if ([[UIDevice currentDevice].systemVersion floatValue] >= 10.0) {
    JPUSHRegisterEntity * entity = [[JPUSHRegisterEntity alloc] init];
    entity.types = UNAuthorizationOptionAlert|UNAuthorizationOptionBadge|UNAuthorizationOptionSound;
    [JPUSHService registerForRemoteNotificationConfig:entity delegate:self];
  } 
  else if ([[UIDevice currentDevice].systemVersion floatValue] >= 8.0) {
    //可以添加自定义categories
    [JPUSHService registerForRemoteNotificationTypes:(UIUserNotificationTypeBadge |
                                                      UIUserNotificationTypeSound |
                                                      UIUserNotificationTypeAlert)
                                          categories:nil];
  } 
  else {
    //categories 必须为nil
    [JPUSHService registerForRemoteNotificationTypes:(UIRemoteNotificationTypeBadge |
                                                      UIRemoteNotificationTypeSound |
                                                      UIRemoteNotificationTypeAlert)
                                          categories:nil];
  }
  //Required
  // 如需继续使用pushConfig.plist文件声明appKey等配置内容，请依旧使用[JPUSHService setupWithOption:launchOptions]方式初始化。
  [JPUSHService setupWithOption:launchOptions appKey:appKey
                        channel:channel
               apsForProduction:isProduction
          advertisingIdentifier:advertisingId];  
```


### 上报DeviceToken

请在AppDelegate.m实现该回调方法并添加回调方法中的代码

```
- (void)application:(UIApplication *)application 
didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
     
  /// Required - 注册 DeviceToken
  [JPUSHService registerDeviceToken:deviceToken];
}

```

### 添加处理APNs通知回调方法（iOS10）

请在AppDelegate.m实现该回调方法并添加回调方法中的代码

```
#pragma mark- JPUSHRegisterDelegate

// iOS 10 Support
- (void)jpushNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(NSInteger))completionHandler {
  // Required
  NSDictionary * userInfo = notification.request.content.userInfo;
  if([notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
    [JPUSHService handleRemoteNotification:userInfo];
  }
  completionHandler(UNNotificationPresentationOptionAlert); // 需要执行这个方法，选择是否提醒用户，有Badge、Sound、Alert三种类型可以选择设置
}

// iOS 10 Support
- (void)jpushNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler {
  // Required
  NSDictionary * userInfo = response.notification.request.content.userInfo;
  if([response.notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
    [JPUSHService handleRemoteNotification:userInfo];
  }
  completionHandler();	// 系统要求执行这个方法
}

```

### 添加处理APNs通知回调方法（iOS10以前）

请在AppDelegate.m实现该回调方法并添加回调方法中的代码

```
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo {
   
  // Required,For systems with less than or equal to iOS6
  [JPUSHService handleRemoteNotification:userInfo];
}
 
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
 
  // Required, iOS 7 Support
  [JPUSHService handleRemoteNotification:userInfo];
  completionHandler(UIBackgroundFetchResultNewData);
}

- (void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error {
  //Optional
  NSLog(@"did Fail To Register For Remote Notifications With Error: %@", error);
}

```

### 添加处理JPush自定义消息回调方法

如需使用JPush的自定义消息功能，请参考[文档](ios_api/message)来实现自定义消息的处理回调方法。


## 成功运行

真机调试该项目，如果控制台输出以下日志则代表您已经集成成功。

```
2016-08-19 17:12:12.745823 219b28[1443:286814]  | JPUSH | I - [JPUSHLogin] 
----- login result -----  
uid:5460310207   
registrationID:171976fa8a8620a14a4  
```

如果调试运行中遇到问题请参考：[iOS SDK 调试指南](ios_debug_guide)

## 高级功能






## Waitting for delete

### 5、创建并配置PushConfig.plist文件 
<div style="font-size:13px;background: #E0EFFE;border: 1px solid #ACBFD7;border-radius: 3px;padding: 8px 16px; padding-bottom: 0;margin-bottom: 0;">
<p>2.1.0 版本开始，新增了带参数的setupWithOption初始化方法，可通过此方法等参数传入AppKey等信息。1.8.8及之前版本的 JPush SDK只能通过PushConfig.plist配置AppKey等信息。
<br/></div>

在你的工程中创建一个新的Property List文件，并将其命名为PushConfig.plist，文件所含字段如下：

* CHANNEL
    * 指明应用程序包的下载渠道，为方便分渠道统计，具体值由你自行定义，如：App Store。
* APP_KEY
    * 填写[管理Portal上创建应用](https://www.jiguang.cn/app/form)后自动生成的AppKey值。请确保应用内配置的 AppKey 与第1步在 Portal 上创建应用后生成的 AppKey 一致。
* APS\_FOR\_PRODUCTION
    * 1.3.1版本新增，用于标识当前应用所使用的APNs证书环境。
    * 0 (默认值)表示采用的是开发证书，1 表示采用生产证书发布应用。
    * 注：此字段的值要与Build Settings的Code Signing配置的证书环境一致。
* 在1.2.2或之前版本的配置文件中，有 TEST_MODE 这个键，新版的SDK不再使用，可以将它删除。

PushConfig.plist文件示例图:

![jpush_ios][2]




#### 集成所需API

APIs 主要集中在 JPUSHService 接口类里。

* 初始化JPush方法分为三个：

    * 1.8.8及以下版本使用的是已过期的初始化方法。升级到2.1.5的老用户仍可继续使用旧的初始化方法。 
    * 2.1.0版本开始提供带appkey等参数的初始化方法。使用此方法无需再添加PushConfig.plist配置JPush的AppKey等字段。
    * 2.1.5版本开始提供带appkey以及IDFA等参数的初始化方法。使用此方法无需再添加PushConfig.plist配置JPush的AppKey等字段。  

	<div style="font-size:13px;background: #E0EFFE;border: 1px solid #ACBFD7;border-radius: 3px;padding: 8px 16px; padding-bottom: 0;margin-bottom: 0;">
<p>使用建议:
<br>
<p>三个初始化 JPush的方法同时存在，以第一个被调用的方法为准。
<br>
	</div>

* JPUSHRegisterDelegate

	为了支持iOS10新特性，在Xcode8及以上开发环境下，2.1.9版本开始需要在类声明中遵守JPUSHRegisterDelegate协议，需要实现其代理方法（具体实现可参考下面代码），并在注册APNs时调用[registerForRemoteNotificationConfig:delegate:]方法设置delegate对象。


```
@interface JPUSHService : NSObject
// init Push
// init Push(<= 1.8.8版本的SDK的注册方法）
+ (void)setupWithOption:(NSDictionary *)launchingOption;

// init Push(2.1.0版本的SDK新增的注册方法)
+ (void)setupWithOption:(NSDictionary *)launchingOption
                 appKey:(NSString *)appKey
                channel:(NSString *)channel
       apsForProduction:(BOOL)isProduction;

// init Push(2.1.5版本的SDK新增的注册方法，改成可上报IDFA，如果没有使用IDFA直接传nil  )
+ (void)setupWithOption:(NSDictionary *)launchingOption
                 appKey:(NSString *)appKey
                channel:(NSString *)channel
       apsForProduction:(BOOL)isProduction
  advertisingIdentifier:(NSString *)advertisingId;

// 注册APNs方法，2.1.9版后可用[registerForRemoteNotificationConfig:delegate:]取代
+ (void)registerForRemoteNotificationTypes:(NSUInteger)types
                                categories:(NSSet *)categories;

// 新版本的注册APNs方法（2.1.9版新增方法，兼容iOS10及以下版本）
+ (void)registerForRemoteNotificationConfig:(JPUSHRegisterEntity *)config delegate:(id<JPUSHRegisterDelegate>)delegate;

// upload device token
+ (void)registerDeviceToken:(NSData *)deviceToken;

// handle notification recieved
+ (void)handleRemoteNotification:(NSDictionary *)remoteInfo;

@end

// iOS 10 Support（2.1.9版新增协议）
@protocol JPUSHRegisterDelegate <NSObject>

// 应用前台运行时，收到本地或远程推送会回调此方法
- (void)jpushNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(NSInteger))completionHandler;
	
// 应用后台运行时，操作通知会回调此方法
- (void)jpushNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler;

@end

```

#### 调用代码

   监听系统事件，相应地调用 JPush SDK 提供的 API 来实现功能。

   以下几个事件监听与调用 JPush SDK API 都是必须的。请直接复制如下代码块里，注释为 "Required" 的行，到你的应用程序代理类里相应的监听方法里。
   
```
  
  return YES;
}
 

 



```


### 7、IDFA
r2.1.5版本增加一个上传IDFA字符串的接口

	 + (void)setupWithOption:(NSDictionary *)launchingOption
                      appKey:(NSString *)appKey
                     channel:(NSString *)channel
            apsForProduction:(BOOL)isProduction
       advertisingIdentifier:(NSString *)advertisingId;
	
如果不使用IDFA，仍可使用接口	
	
	+ (void)setupWithOption:(NSDictionary *)launchingOption
                      appKey:(NSString *)appKey
                     channel:(NSString *)channel
            apsForProduction:(BOOL)isProduction;
            
### 8、监听通知

建议开发者加上API里面提供下面 5 种类型的通知：

extern NSString *const kJPFNetworkIsConnectingNotification; // 正在连接中

extern NSString * const kJPFNetworkDidSetupNotification; // 建立连接

extern NSString * const kJPFNetworkDidCloseNotification; // 关闭连接

extern NSString * const kJPFNetworkDidRegisterNotification; // 注册成功

extern NSString *const kJPFNetworkFailedRegisterNotification; //注册失败

extern NSString * const kJPFNetworkDidLoginNotification; // 登录成功

<div style="font-size:13px;background: #E0EFFE;border: 1px solid #ACBFD7;border-radius: 3px;padding: 8px 16px; padding-bottom: 0;margin-bottom: 0;">
<p>温馨提示：
  <br>
<p>Registration id 需要添加注册kJPFNetworkDidLoginNotification通知的方法里获取，也可以调用[registrationIDCompletionHandler:]方法，通过completionHandler获取
</div>

extern NSString * const kJPFNetworkDidReceiveMessageNotification; // 收到自定义消息(非APNs)

其中，kJPFNetworkDidReceiveMessageNotification传递的数据可以通过NSNotification中的userInfo方法获取，包括标题、内容、extras信息等

### 高级功能

请参考文档：[iOS SDK API](http://docs.jpush.io/client/ios_api/)


### 技术支持

邮件联系：[support@jpush.cn][4]

问答社区：[http://www.jpush.cn/qa/][5]

[0]: ../image/create_ios_app.jpg
[1]: ../image/Screenshot_13-4_2_create.jpg
[2]: ../image/Screenshot_13-4-15_3_31.png
[3]: ios_api
[4]: mailto:support@jpush.cn
[5]: http://community.jiguang.cn
[6]: ../image/ios_http.png
[7]: ../image/capabilities_intro.jpg