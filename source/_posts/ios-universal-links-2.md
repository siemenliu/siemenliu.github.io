---
title: iOS App间相互跳转漫谈 - part2
date: 2017-09-10 14:07:20
tags: 
- iOS
- UniversalLinks
- Google
- DynamicLinks
---

## 写在前面

[上一篇](http://www.liuxiaoming.com/2017/09/07/ios-universal-links/)已经介绍了iOS系统层面提供的应用之间跳转的技术和实施方案，本篇会带大家更深入的了解UniveralLinks技术，探究其绑定运作原理，使用时的技巧。

## 运作原理

对于UniversalLinks的生效和绑定原理，官方文档并未说明，通过亲自尝试整理出了其绑定原理，这有助于对于触发时机把握：

1. App从iOS9开始，安装App成功后，系统会检查App是否对UniversalLinks技术做了支持，如果检查到有做支持，则从App的associated-domains配置里读取绑定域名。

2. 访问该域名下预设的JSON文件`apple-app-site-association`，读取JSON文件中的app绑定信息，与当前App内的信息做对比。

3. 如果匹配成功，在系统本地内部就做完了双向绑定。

也就是只要App安装完成，就可以即刻触发UniversalLinks打开App啦！不需要像推送那样，需要打开一次。

## 巧用UniveralLinks判断App是否安装

上一篇文章也提到了，目前所有包括UniversalLinks技术都无法直接对App的安装做判断。

但我要说：非也！UniversalLinks提供的一项特性可以让我们反向推断App是否安装，而且准确率相当高，不像scheme跳转那样，只能做延时。

UniversalLinks在触发时，也就是链接被点击时，系统会与本地已经建立双向绑定的数据进行匹配，匹配到的话，这个网络请求就会在系统层面被拦截，系统就会转而打开绑定的App，并把完整URL传给App的openURL的Delegate方法来处理。反之，如果App没有安装，那么点击链接时系统无法匹配到信息，则就将请求放行，让服务端接收到这个请求来处理。

看到这里已经很明了了吧，用户只要点击了，服务端可以通过是否接收到请求来判断App是否已安装。

当然这里也会不准，但概率比较小，原因如果用户是长按打开，或者打开App后点击了右上角的系统的返回按钮后，系统会下次触发UniversalLinks的时候就不会拦截请求了，导致接收到请求的设备其实已经App，而我们无法知道。要想重新让系统拦截，用户需要点击的时候长按link，并且选择通过App打开，之后才又会被拦截。

但已经比scheme跳转设置一个timeout要来的靠谱的多了。

## 巧用UniveralLinks拉新引旧

UniversalLinks作为一个平滑使用体验的工具类技术来说，本身不具备拉新客户的功能：比如新用户如果从站外点击UniversalLinks，那么用户没有装App的话，只是会访问m站而已，无它；引导老用户从H5迁移到App的能力也比较弱，一直使用m站的用户，只有在页面顶部看到有一个入口，点击才能打开App。原因是用户如果直接输入了m站的地址，在站内访问，是不会触发UniversalLinks的，只是会在绑定的H5页面顶部有一个bar提醒用户，点击可以跳转到App的相同功能，可以说是很弱的。

让我们看看怎么才能让它具备强制拉新引旧能力：

- 首先要具备判断是否安装App的条件来区分新用户还是老用户，如果未安装则为新，已安装则为旧。方法上面一节已经介绍。
- 接下来服务端开发一个特殊的UniversalLinks，类似于https://app.alibaba.com/babalink，对app做好双向绑定，注意，这里域名与m站的m.alibaba.com域名必须不同。
- babalink接收两种参数scheme=<scheme URL>&url=<http URL>，前者是应用的scheme跳转，服务端拼接传入打开App后需要打开页面的scheme值，这个值是给已安装App的情况用的；还有url中传入需要跳转的http页面，通常是这个链接原本功能的m站链接。
- 这里举个栗子，例如跳转到产品详情，app的scheme是enalibaba://detail?id=123；而m站的页面是https://m.alibaba.com/product?id=123；那么服务端在搜索结果页每个结果的链接应该是`https://app.alibaba.com/babalink?scheme=enalibaba://detail?id=123&url=https://m.alibaba.com/product?id=123`每个一级参数下的值都需要urlencode，这边为了查看方便就不做了。
- 这样，用户到了m站搜索结果页的时候，每个搜索结果点击的都是上面这种链接，这种链接的域名与m站本身域名不同，在App已安装的情况下就会触发系统拦截请求，跳转App，App在打开时通过完整url中的scheme参数，进行页面跳转即可完成引导老用户的操作。
- 再看看新用户，新用户因为没有安装App，所以这个link的请求会被发送到服务端，服务端根据情况控制将其引导到AppStore去安装App，或者直接重定向到url参数的网页。后者不用说还是延续用户在m站的上的继续浏览，前者打开了AppStore安装了App后，用户打开时首页怎么办？
- 这里这里可以利用归因系统来做到从AppStore打开，还能继续之前的操作，大致原理是服务端接收到link的请求，从请求中将这个请求转化为用户指纹存起来。
- 当用户打开刚安装好的App后，App会在启动的时候将一些可作为指纹的数据当做参数通过某个接口传给服务端，服务端接收App的指纹数据参数后，通过算法匹配之前的用户指纹，匹配上后将用户点击的完整链接返回给客户端，由客户端来讲链接解析，利用其scheme值进行跳转，即可完美continue用户的行为。

## 从其他App直接打开

- 从浏览器打开，只要是当前页面的域名和用户点击按钮的universallinks的域名是不同的，就会触发。
- 在App中打开会有些麻烦，只有通过`[[UIApplication sharedApplication] opentURL:@"https://app.alibaba.com/babalink?scheme=enalibaba://detail?id=123&url=https://m.alibaba.com/product?id=123"]`这样打开，才能跳转到目标app，如果把链接塞到webview中则不会触发，请求一定会到服务端。
- 所以，微信里直接通过聊天内容发送链接跳转是做不到的。
- 这里可以让链接检测浏览器UA来判断链接是否为从其他App打开，如果是，则重定向到`https://app-c.alibaba.com/babalink?scheme=enalibaba://detail?id=123&url=https://m.alibaba.com/product?id=123`注意这个链接的域名不是`app.alibaba.com`，这个h5页面展示一个按钮，跳转链接为`https://app.alibaba.com/babalink?scheme=enalibaba://detail?id=123&url=https://m.alibaba.com/product?id=123`。为什么？因为要跨域，而且点击行为在浏览器中进行是无法阻断的。
- 此时用户点击中间页按钮时，即便在其他App，不管是Facebook还是微信，都可以触发UniversalLink啦！

## 总结

总结一下，巧用UniversalLink需要让服务端的这个link具备

1. 继续用户m站行为
2. 跳转AppStore行为
3. 重定向到中间页行为
4. 链接归因存储匹配能力

