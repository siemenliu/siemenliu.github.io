---
title: App网络实时数据模拟方案
date: 2017-08-21 23:57:22
tags: iOS, Andoird, Network, Mock, RealTime
---

## 背景

客户端同学与服务端同学进行联调时，总是会遇到各种各样的问题。其中，以服务端的线下环境不稳定是最为头疼，联调过程中，一会应用要重启了，一会某个依赖的公共线下环境挂了，或者为了让服务端增加某个字段需要等半天才行。

这些直接导致的结果就是，联调开发效率非常低下！陷入了各种相互等的循环造成了各种碎片的垃圾时间。

于是就想想办法让客户端同学开发的时候尽量少依赖，甚至不依赖服务端的运行环境，这样就可以完美高效开发了。

## 原始方案

最开始的做法也是相当普通的做法，就是把服务端的内容结果json字符串以json文件的形式加入app安装包中，app启动后会根据app当前的网络情况判断是获取本地的json文件中的内容还是直接获取真实接口内容。

这种做法其实已经很大程度的解决了最核心问题，如果是这个方案我不需要专门写一篇文章来介绍。

## 有人吐槽

产品的不断迭代其实就是一直被吐槽，被吐槽说明被需要，有人用，只是觉得不爽。

有了原始方案后，客户端同学们已经大量从服务端依赖中解救出来，陷入了新的烦恼，原因是只要需要对返回结果做一些哪怕是一点点改动，都需要重新编译压包，这是很恼火的事情。

我自己开发过程中也深刻体会到了这一点，并也开始思考，怎么样不重新编译压包就可以修改mock数据。

思考了几个方案：

1. 代理：使用Charles，通过map功能将请求返回结果通过map到的本地文本文件返回，就可以完美实现实时修改mock结果。但需要查看电脑ip，还要手机设置代理，还要配置ssl证书等一堆破事，作为程序员总觉的还是要操作很多步骤非常不爽。
2. 远程修改安装包中的mock文件内容：通过本机命令行操作模拟器和真机，读写app安装包中的mock文件内容来达到目的，这条路探索了一阵子没有找到落地的方案。

## 新的思路

那天的一件事突然激发了灵感，想出了目前最完美的方案。那天具体目的是啥忘了，总之想要在本机开启一个http静态服务器，设置到任意一个本地目录为root，把里面的内容可以被别人下载访问，看了下mac自带的Apache，需要设置虚拟机等配置，并不是很方便，没法达到我最简单最纯粹的目的。

最终还是探索出了一行命令在任意目录开启http服务的途径，后面会详细介绍。

由此突然想到，如果我把静态服务器root置为项目目录下的mock文件夹，再将本机ip让app感知到，那在运行的时候就可以将请求转发给ip指向的机器，来做到实时mock数据的功能！

## 全新设计方案

由此设计出一套不需要任何辅助app支援，不需要手机设置代理，而且可以不用重新编译就可以实时修改mock数据的网络数据模拟方案。

我本身是开发iOS所以下面实施方案都以iOS为例，安卓上思路可以参考。

### 1. 让app知道哪台机器开启了mock数据的http服务
   说的明白一点就是让编译的app知道当前压包的机器是什么ip，这个思路就很多了，我也是采用了最普通的做法，通过在编译过程中插入脚本，读取本机ip，写入当前app的Info.plist文件。

```bash 一行命令获取本机ip
ifconfig | grep "inet " | grep -Fv 127.0.0.1 | awk '{print $2}'
```
   
将获得ip写入Info.plist
   
```bash 写入数据到plist
plist="${PROJECT_DIR}/${INFOPLIST_FILE}"
key="XM_MOCK_IP"
value="192.168.2.101"
/usr/libexec/Plistbuddy -c "Delete :${key}" "${plist}"
/usr/libexec/Plistbuddy -c "Add :${key} string ${value}" "${plist}"
```

### 2. 在编译机上工程项目Mock目录下启动http静态服务

脚本cd到Mock目录下，执行下面命令就可以。

```bash 在当前目录开启http服务
lsof -ti:7777 | xargs kill
SCREEN -d -m python -m SimpleHTTPServer 7777
```

第一行命令清理一下占用7777端口的应用。
第二行命令可以以当前执行命令的目录为root开启一个7777端口的静态服务器，这句话试了目前主流的macOS版本都支持这样启动。

ps. 解释下Mock目录，Mock目录我的设计是目录中存放的是所有以apiname.json为文件名的请求结果字符串文本文件。
这个目录存放在项目的Resource目录下，直接以物理目录在Xcode下可见。
目的是为了在开发过程中方便随时新增和修改内容，不需要再切换其他编辑器来修改。

   
### 3. 让app发起网络请求时转发到mock服务器

这里没什么代码可以提供，大体就是app上得有个开关，可以随时开关app的mock和非mock状态，根据这个标志位，网络请求生成的请求URL会有所不同：

- 在mock模式下
  生成的请求url会变成`http://192.168.2.101:7777/xmmock/xxx.json`
  网络请求正常发出，或得到的内容就是mock内容，此时修改macOS里Xcode项目中mock文件夹下的json文件内容就可以实时生效修改。
  这里的`xmmock/xxx.json`就是root目录下的mock文件结构，可根据规则动态生成。
  
- 在非mock模式下
  正常生成url请求
  
  
### 4. 局部数据模拟 
  
这里还可以做一下容错，大体是如果mock模式下，请求`http://192.168.2.101:7777/xmmock/xxx.json`后如果是非200，说明这个mock文件不存在，则继续发送正常请求，以确保没有在mock范围内的接口在app中还是可以正常访问。

这样就做到了app接口整体可用，只有在mock模式下，在mock范围内的接口会被mock。

### 5. 清理尾巴

这里有点小尾巴，比如之前生成的在plist文件中加入了一个键值标明当前压包的机器ip，这个键值在git中显示是一个修改，这样每个人修改这个代码很容易冲突。

这个也很好办，在写入编译包后，编译完成的环节再把这个键值从plist中自动移除即可

```bash
plist="${PROJECT_DIR}/${INFOPLIST_FILE}"
/usr/libexec/Plistbuddy -c "Delete :XM_MOCK_IP" "${plist}"
```

## 全新的联调流程

做到这样以后，可以想象一下，在项目进入开发后，客户端和服务端同学只要事先约定好接口各种情况数据返回的格式，客户端就可以利用这套方案直接当做服务端已经提供接口那样进行开发，当客户端通过本mock方案联调自测各种情况通过后，再与服务端真实接口对接，成本可以说是非常的小，完全消除的之前的相互等待的垃圾碎片时间。

目前我们团队的iOS组内已经使用这套方案成熟运行了好几个版本，开发效率提升明显。


## 附上完整脚本

将以下deploy脚本插入到"Build Phases"里尽可能前面的环节，实际操作总我放在"Copy Bundle Resources"的前面。

```bash XMNetworkMockDeploy.sh
#!/bin/sh
## 给plist赋值函数
setValueToPlist() {
    plist="${PROJECT_DIR}/${INFOPLIST_FILE}"
    
    key="$1"
    value="$2"
    /usr/libexec/Plistbuddy -c "Delete :${key}" "${plist}"
    /usr/libexec/Plistbuddy -c "Add :${key} string ${value}" "${plist}"
    echo "XMNetworkMock::在PLIST设定${key}值为${value}"
}

echo "XMNetworkMock::CONFIGURATION:${CONFIGURATION}"
echo "XMNetworkMock::PROJECT_DIR:${PROJECT_DIR}"

if [[ $CONFIGURATION == "Debug" ]]; then

## 获取主机IP地址
ASB_HOST_IP=`ifconfig | grep "inet " | grep -Fv 127.0.0.1 | awk '{print $2}'`
setValueToPlist "XM_MOCK_IP" $ASB_HOST_IP
echo "XMNetworkMock::主机IP地址${ASB_HOST_IP}"

## 清理端口进程
lsof -ti:7777 | xargs kill
echo "XMNetworkMock::端口占用清理完毕"

## 进入目录 
ASB_DIR_MOCK_API="${PROJECT_DIR}/Resources/xmmock"
mkdir -p $ASB_DIR_MOCK_API
echo "XMNetworkMock::打开目录${ASB_MOCK_API}"

## 开启网络服务
cd $ASB_DIR_MOCK_API
SCREEN -d -m python -m SimpleHTTPServer 7777
echo "XMNetworkMock::启动网络服务 ${ASB_DIR_MOCK_API}:7777"

else
echo "XMNetworkMock::检测到非开发模式，已清理调试选项ASC_HOST_IP的值"
fi
```

以下clean脚本放置在"Build Phases"的最后一个流程即可

```bash XMNetworkMockClean.sh
#!/bin/sh
deleteValue() {
plist="${PROJECT_DIR}/${INFOPLIST_FILE}"
key="$1"
/usr/libexec/Plistbuddy -c "Delete :${key}" "${plist}"
}
if [[ $CONFIGURATION == "Debug" ]]; then
deleteValue "XM_MOCK_IP"
fi
```

ps. 注意脚本中相关目录。

