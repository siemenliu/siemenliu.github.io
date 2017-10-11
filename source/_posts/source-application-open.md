# iOS如何识别App打开的来源

## 写在前面

在数据统计方面层面看App打开来源非常重要，特别是对于分享以及付费引流的衡量效果上有着非常关键的作用。

iOS App最常用的打开途径有三种：

1. 消息（本地、远程）推送打开
2. scheme跳转打开
3. UniversalLinks打开

## 如何区分

App启动后标志着App底层已经准备完备的地方就有可以用来区分的标志，也就是在AppDelegate中的`application:didFinishLaunchingWithOptions:`

``` Objective-C
if (launchOptions[UIApplicationLaunchOptionsRemoteNotificationKey]) {
    NSLog(@"远程推送打开");
} else if (launchOptions[UIApplicationLaunchOptionsLocalNotificationKey]) {
    NSLog(@"本地推送打开");
} else if (launchOptions[UIApplicationLaunchOptionsUserActivityDictionaryKey]) {
    NSLog(@"UniversalLinks打开");
} else if (launchOptions[UIApplicationLaunchOptionsURLKey]) {
    NSLog(@"Scheme跳转打开");
} else if (!launchOptions) {
    NSLog(@"手动点击打开");
}
```

## 再多说一些

launchOptions除了用来区分App的开发方式，还承载着打开时的一些数据，比如scheme跳转、UniversalLinks打开的时候的一些具体链接，之前应用的bundleID等数据方便追述。

例如UniversalLinks中，我们就可以通过如下方法获得链接，而不一定要等到专用的Delegate方法返回给我们

```
NSUserActivity *act = [[launchOptions objectForKey:UIApplicationLaunchOptionsUserActivityDictionaryKey] objectForKey:@"UIApplicationLaunchOptionsUserActivityKey"];
NSString *url = [act.webpageURL absoluteString];
```

## 其他枚举的意义

- UIApplicationLaunchOptionsURLKey
   在scheme跳转打开的时候出现，用于获取scheme地址的key；如"enalibaba://home"

- UIApplicationLaunchOptionsSourceApplicationKey
   从其他App跳转打开的时候出现 值为一个字符串表示来源App的BundleID；如："com.apple.mobilesafari" 表示从Safari跳转

- UIApplicationLaunchOptionsRemoteNotificationKey
   从远程推送打开，这个key对应的值是一个Dictionary，里面就是推送的Payload

- UIApplicationLaunchOptionsLocalNotificationKey
   从本地推送打开，这个key对应的值是一个Dictionary，里面就是推送的Payload

- UIApplicationLaunchOptionsAnnotationKey
   这个Key应该不会再看见了，它只能通过 `application:openURL:sourceApplication:annotation:` 打开App的时候才会出现，但这个方法已经被标记为 Deprecated 从9.0之后不再支持。

- UIApplicationLaunchOptionsLocationKey
   基于地理位置触发的App打开，官方文档应该更新过了，已经找不到原文，大意是如果App开启了地理位置，在App退出到后台之后，如果触发了地理位置打开App，那么LaunchOptions就会有这个Key，可用来开启地理位置事件监听的标志，但这种地理位置触发打开App的能力需要App Store审核才能开启使用。

- UIApplicationLaunchOptionsNewsstandDownloadsKey
   关于杂志更新的，用到极少，不多描述

- UIApplicationLaunchOptionsBluetoothCentralsKey
   蓝牙服务提供设备唤醒App时出现的Key，数据为数组，代表设备列表

- UIApplicationLaunchOptionsBluetoothPeripheralsKey
   蓝牙被服务设备唤醒App时出现的Key，数据为数组，代表设备列表

- UIApplicationLaunchOptionsShortcutItemKey
   从3D Touch打开App时出现此Key

- UIApplicationLaunchOptionsUserActivityDictionaryKey
   UniversalLinks打开时出现此Key，用于获得继续打开行为的一些数据，通常是点击的链接

- UIApplicationLaunchOptionsUserActivityTypeKey
   UniversalLinks打开时出现此Key，值为NSUserActivityTypeBrowsingWeb，而且这个枚举值就这个以

- UIApplicationLaunchOptionsCloudKitShareMetadataKey
   A key indicating that the app received a CloudKit share invitation. 相关文档比较少，推测是从文件里分享给App打开时会出现此Key。

参考
https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623073-application?language=objc
http://nshipster.cn/launch-options/
http://www.jianshu.com/p/2ab2716c334e
http://www.jianshu.com/p/6a1eb76ec776

