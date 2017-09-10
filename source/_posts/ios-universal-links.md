---
title: iOS App间相互跳转漫谈 - part1
date: 2017-09-07 23:48:26
tags: iOS,UniversalLinks,Google,DynamicLinks
---

## 概述

文章介绍当下iOS系统中各种App之间的跳转技术，并最终重点介绍UniversalLinks的一种特殊的使用技巧来帮助App来引流，提升转化。

## 背景

介绍下当下支持的App页面跳转技术及其优劣：

1. Scheme跳转
   例如：
   ```
   <appscheme>://detail?id=10000
   ```
   用户在系统中其他App中点击scheme链接；浏览器网页中点击scheme链接会弹出一个Alert弹框，让用户确认是否跳转。
   优势：与http的url提供类似，可以通过URL直观表达跳转的页面和意思。触发条件可以是用户点击，也可以通过程序触发JS或者App。
   劣势：跳转时系统会弹出确认框让用户确认，体验略差。并且不能知道App是否安装，只能通过一些手段推测需要跳转的App是否已经安装，如果跳转时没有安装则会弹出“Safari不能打开该网页，因为网址无效。”的提示体验打折扣，关于H5页面如何推测App已经安装后面会介绍。
   
2. Smart App Banners
   在HTML页面中植入一个meta标签，类似于：
   ```
   <meta name="apple-itunes-app" content="app-id=myAppStoreID, affiliate-data=myAffiliateData, app-argument=myURL">
   ```
   访问这个页面的时候在网页加载完后往下拉页面可以看到App的跳转入口，可以承接跳转的App的功能。
   可以手机访问这个[Demo页面](http://www.liuxiaoming.com/demo/app-smart-banner.html)来体验下
   优势：对用户而言，点击进入不需要二次确认；自动检测App安装状态，未安装引导到App Store，已安装则可直接打开App，并带入预置在app-argument中的url值，让App感知当前的页面，可以App中继续行为。
   劣势：最大的问题是跳转操作不可通过程序控制，只能由用户点击触发跳转；另一个产品经理不怎么喜欢的是居然还显示评分星级，这是个头疼的事情。
   
3. Univeral Links
   从iOS9开始，苹果开放了一种新的App之间的跳转方式，从使用体验上来看，苹果希望这种技术用于不打断用户的操作行为为前提，提供更好的App跳转方式。举例来说，正在访问用户正在访问m.alibaba.com，那么如果用户已经安装了Alibaba.com App那么系统将自动引导到App，并将正在访问的完整http链接提供到打开的App以提供在App上继续之前行为的能力。
   这种技术带来了完美的体验，通俗点说，现在还能提供http的URL(当然需要https)直接打开App，而且没有二次确认！当然也有些地方有待完善，这个下文会详细提到。
   
> 三种技术各有优劣，组合使用可以让引流体验变得非常好。下面会简单介绍三种技术的部署方法和一些技巧。

## 部署

### Scheme方式跳转

这种技术部署是需要App本身对scheme协议头进行注册的，而且这种注册是属于系统级别本地注册关联。

1. 项目的Info.plist文件中，新增一个Key -> "URL Types"预置的Key输入时有联想。
2. 新增完后会自动变成一个数组，展开查看第一个元素，将其中 "URL identifier" 置为App的BundleID
3. 并且平级新增一个Key -> "URL Schemes"预置Key有联想。
4. 新增完后自动变成数组，展开，在第一个元素中填入想要注册的scheme协议头即可，比如想要注册 "enalibaba://"，那么只要输入 "enalibaba" 即可。

这时App安装后，协议头即注册生效，可以通过浏览器地址栏中输入该协议头即可跳转。

注意：因为是本地注册，所以如果遇到某个其他App也注册这个协议头那会发生什么情况呢？经过实践测试结果发现，协议头是本地抢注式，先被安装的注册的App拥有更高的优先权，当先装的App被删除时，第二注册者会命中，以此后推。

### App Smart Banner

这种方式是直接去中心化，去校验，完全自由的一种部署方式。

唯一需要做的只是在html页面新增meta标签即可

```
<meta name="apple-itunes-app" content="app-id=myAppStoreID, affiliate-data=myAffiliateData, app-argument=myURL">
```

使用iOS手机访问这个地址 [Demo页面](http://www.liuxiaoming.com/demo/app-smart-banner.html)

### Universal Links

这种方式更为麻烦一些，因为相当于支持了http链接直接打开App，所以需要双向授信才行。

网站端
1. 创建一个名为`apple-app-site-association`的JSON格式文件，防止在可以被根目录直接访问到的地方。例如打算绑定m.alibaba.com那么就需要`https://m.alibaba.com/apple-app-site-association`这个文件可以正常访问，mine/type 为 json/text。
2. 文件的内容格式如下
   ```JSON
    {
        "applinks": {
            "apps": [],
            "details": [
                {
                    "appID": "9JA89QQLNQ.com.apple.wwdc", //bundleid
                    "paths": [ "/wwdc/news/", "/videos/wwdc/2015/*"] //path 可以通配符
                },
                { // 可配置多个域名和path匹配规则的绑定
                    "appID": "ABCD1234.com.apple.wwdc",
                    "paths": [ "*" ]
                }
            ]
        }
    }
   ```

App端
1. 找到 AppName.entitlements 文件
2. 在里面 `com.apple.developer.associated-domains` 的数组下面添加一个item
3. 值为 `applinks:m.alibaba.com` 即可

当然App端能找到entitlements文件需要打开Capabilities中的Associated Domain能力，这需要开发者账号项目才能开启。

到这里双向绑定已做完，编译项目到设备上，通过点击在 备忘录 中预置的匹配Universal Links链接，即可完美跳转了。

为什么要放到备忘录？不能直接浏览器地址栏打开吗？答案是不行，还记得上面讲过，苹果希望的是不打断用户意图的情况下，所以在浏览器地址栏中直接输入地址的行为，苹果认为用户有明显意图想要访问http网页，所以不会直接跳转到App。当然程序也无法控制，如果是JS程序跳转某个UniversalLinks，那结果就是请求正常发出，并不会跳转到App。

而且，而且，苹果的初衷其实是希望不同App之间可以通过这种方式相互跳转。
当初我亲测微信跳转Alibaba.com App完美跳转。
后来，有一种非常规操作可以阻挡这种跳转，以至于现状是几乎所有App都禁用了UniversalLinks的外跳。
但我仍然有解决办法，我会把UniversalLinks的运作原理和攻防技巧放在下一篇文章里详细描述。
   
   
## 豆知识

### Scheme和Schema

其实正确的叫法应该叫Scheme[skim]，而不是Schema['ski:mə]，[官方文档](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html)在描述中也是一直使用Scheme。

从词典上查看，这两个词的意思表达非常相近，都有计划策划的意思。

## 参考资料

[Promoting Apps with Smart App Banners](https://developer.apple.com/library/content/documentation/AppleApplications/Reference/SafariWebContent/PromotingAppswithAppBanners/PromotingAppswithAppBanners.html)

[Inter-App Communication](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html)

[AppID查找工具](https://linkmaker.itunes.apple.com/zh-cht)

[Support Universal Links](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html)

