---
layout:     post
title:      "iOS开发中WiFi相关功能总结"
date:       2016-05-17
author:     "Haley_Wong"
catalog:    true
tags:
    - WIFI
---

iOS 开发中难免会遇到很多与网络方面的判断，这里做个汇总，大多可能是与WiFi相关的。

# 1.Ping域名、Ping某IP

有时候可能会遇到ping 某个域名或者ip通不通，再做下一步操作。这里的ping与传统的做get或者post请求还是有很大区别的。比如我们连接了某个WiFi，测试ping www.baidu.com，如果能ping 通，基本可以断定可以上网了，但是如果我们做了一个get 请求（url 是www.baidu.com）,路由器可能重定向这个WiFi内的某网页了，依然没有错误返回，就会误认为可以正常上网。

这里有关于ping命令的详细解释：[百度百科Ping](http://baike.baidu.com/link?url=oVrNiOv5BJT8AgQNjvYT0ZjC5xsFPsvtLzcu-1hPLtgVSmfXhZpQYorHETRqHKtfY47RTEq2giOWSMT6WnbZQ9DIKaoJtDjT8G811SznCYi)

iOS中想要ping域名或者ip，苹果提供了一个官方例子[SimplePing](https://developer.apple.com/library/mac/samplecode/SimplePing/Introduction/Intro.html#//apple_ref/doc/uid/DTS10000716-Intro-DontLinkElementID_2)

在例子中，有一个苹果已经封装过的类【SimplePing.h】和【SimplePing.m】

使用起来也相当的简单：

* 首先创建一个Ping对象：
```
    SimplePing *pinger = [[SimplePing alloc] initWithHostName:self.hostName];
    self.pinger = pinger;
    
    pinger.delegate = self;
    pinger.addressStyle = SimplePingAddressStyleICMPv4;
    [pinger start];
```

* 然后在start成功的代理方法中，发送数据报文：

```
/**
 *  start成功，也就是准备工作做完后的回调
 */
- (void)simplePing:(SimplePing *)pinger didStartWithAddress:(NSData *)address
{
    // 发送测试报文数据
    [self.pinger sendPingWithData:nil];
}

- (void)simplePing:(SimplePing *)pinger didFailWithError:(NSError *)error
{
    NSLog(@"didFailWithError");
    [self.pinger stop];
}
```

其他几个代理方法也非常简单，就简单记录一下：

```
// 发送测试报文成功的回调方法
- (void)simplePing:(SimplePing *)pinger didSendPacket:(NSData *)packet sequenceNumber:(uint16_t)sequenceNumber
{
    NSLog(@"#%u sent", sequenceNumber);
}

//发送测试报文失败的回调方法
- (void)simplePing:(SimplePing *)pinger didFailToSendPacket:(NSData *)packet sequenceNumber:(uint16_t)sequenceNumber error:(NSError *)error
{
    NSLog(@"#%u send failed: %@", sequenceNumber, error);
}

// 接收到ping的地址所返回的数据报文回调方法
- (void)simplePing:(SimplePing *)pinger didReceivePingResponsePacket:(NSData *)packet sequenceNumber:(uint16_t)sequenceNumber
{
    NSLog(@"#%u received, size=%zu", sequenceNumber, packet.length);
}

- (void)simplePing:(SimplePing *)pinger didReceiveUnexpectedPacket:(NSData *)packet
{
    NSLog(@"#%s",__func__);
}
```
> 注意点：
iOS 中 ping失败后（即发送测试报文成功后，一直没后收到响应的报文）,不会有任何回调方法告知我们。而一般ping 一次的结果也不太准确，ping 花费的时间也非常短，所以我们一般会ping多次，发送一次ping 测试报文0.5s后检测一下这一次ping是否已经收到响应。0.5s后检测时，如果已经收到响应，则可以ping 通；如果没有收到响应，则视为超时。
做法也有很多种，可以用`NSTimer`或者 `{- (void)performSelector: withObject:afterDelay:}`

这里有一个别人写的工程[https://github.com/lovesunstar/STPingTest](https://github.com/lovesunstar/STPingTest)

![PingTest效果图](/img/blogs/ios_wifi/img_01.webp)

![终端ping效果图](/img/blogs/ios_wifi/img_02.webp)

# 2.获取WiFi信息
以前物联网刚火的时候，出现过很多一体式无线路由，所以App里难免会遇到要判断当前所连接的WiFi，以及获取WiFi信息的功能。13年的时候查过一些关于WiFi的方法，后面渐渐都忘记了。

需要添加`SystemConfiguration.framework`  并在当前类中添加代码`#import <SystemConfiguration/CaptiveNetwork.h>`

```
//获取WiFi 信息，返回的字典中包含了WiFi的名称、路由器的Mac地址、还有一个Data(转换成字符串打印出来是wifi名称)
- (NSDictionary *)fetchSSIDInfo {
    NSArray *ifs = (__bridge_transfer NSArray *)CNCopySupportedInterfaces();
    if (!ifs) {
        return nil;
    }
    
    NSDictionary *info = nil;
    for (NSString *ifnam in ifs) {
        info = (__bridge_transfer NSDictionary *)CNCopyCurrentNetworkInfo((__bridge CFStringRef)ifnam);
        if (info && [info count]) { break; }
    }
    return info;
}

//打印出来的结果：
2016-05-12 15:28:51.674 SimplePing[18883:6790207] WIFI_INFO:{
    BSSID = "a4:2b:8c:c:7f:bd";
    SSID = bdmy06;
    SSIDDATA = <73756e65 65653036>;
}
```

# 3.获取WiFi名称
有了上一步，获取WiFi名称就非常简单了。

```
NSString *WiFiName = info[@"SSID"]; 

//打印结果：
2016-05-12 15:35:13.059 SimplePing[18887:6791418] bdmy06
```

完整的：
```
- (NSString *)fetchWiFiName {
    NSArray *ifs = (__bridge_transfer NSArray *)CNCopySupportedInterfaces();
    if (!ifs) {
        return nil;
    }
    NSString *WiFiName = nil;
    for (NSString *ifnam in ifs) {
        NSDictionary *info = (__bridge_transfer NSDictionary *)CNCopyCurrentNetworkInfo((__bridge CFStringRef)ifnam);
        if (info && [info count]) {
            // 这里其实对应的有三个key:kCNNetworkInfoKeySSID、kCNNetworkInfoKeyBSSID、kCNNetworkInfoKeySSIDData，
            // 不过它们都是CFStringRef类型的
            WiFiName = [info objectForKey:(__bridge NSString *)kCNNetworkInfoKeySSID];
//            WiFiName = [info objectForKey:@"SSID"];
            break;
        }
    }
    return WiFiName;
}
```

# 4.获取当前所连接WiFi的网关地址
例如自己家的路由器一般默认的网关地址是192.168.1.1，获取的就是这个192.168.1.1。

为什么不直接写死呢？

因为一些商场或者有多个路由器的网关地址是不一样的，比如之前有个公司的网关是192.168.89.1。
这里有篇博客，[这是地址](http://blog.csdn.net/u010990519/article/details/41677549)

需要导入的库：
```
#import <arpa/inet.h>
#import <netinet/in.h>
#import <ifaddrs.h>
```
获取网关的方法
```
- (NSString *)getGatewayIpForCurrentWiFi {
    
    NSString *address = @"error";
    struct ifaddrs *interfaces = NULL;
    struct ifaddrs *temp_addr = NULL;
    int success = 0;
    
    // retrieve the current interfaces - returns 0 on success
    success = getifaddrs(&interfaces);
    if (success == 0) {
        // Loop through linked list of interfaces
        temp_addr = interfaces;
        //*/
        while(temp_addr != NULL) {
        /*/
         int i=255;
         while((i--)>0)
         //*/
            if(temp_addr->ifa_addr->sa_family == AF_INET) {
                // Check if interface is en0 which is the wifi connection on the iPhone
                if([[NSString stringWithUTF8String:temp_addr->ifa_name] isEqualToString:@"en0"])
                {
                    // Get NSString from C String //ifa_addr
                    //ifa->ifa_dstaddr is the broadcast address, which explains the "255's"
                    //                    address = [NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_dstaddr)->sin_addr)];
                    
                    address = [NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_addr)->sin_addr)];
                    
                    //routerIP----192.168.1.255 广播地址
                    NSLog(@"broadcast address--%@",[NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_dstaddr)->sin_addr)]);
                    //--192.168.1.106 本机地址
                    NSLog(@"local device ip--%@",[NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_addr)->sin_addr)]);
                    //--255.255.255.0 子网掩码地址
                    NSLog(@"netmask--%@",[NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_netmask)->sin_addr)]);
                    //--en0 端口地址
                    NSLog(@"interface--%@",[NSString stringWithUTF8String:temp_addr->ifa_name]);
                    
                }
            }
            
            temp_addr = temp_addr->ifa_next;
        }
    }
    
    // Free memory
    freeifaddrs(interfaces);
    
    in_addr_t i = inet_addr([address cStringUsingEncoding:NSUTF8StringEncoding]);
    in_addr_t* x = &i;
    
    unsigned char *s = getdefaultgateway(x);
    NSString *ip=[NSString stringWithFormat:@"%d.%d.%d.%d",s[0],s[1],s[2],s[3]];
    free(s);
    return ip;
}
```
其中 getdefaultgateway 是一个C语言文件中的方法，在工程里可以找到。

>  很多人说找不到getgateway.h的地址，下载地址在这里：[工程地址](https://download.csdn.net/download/u011619283/11052780)

# 5.获取本机在WiFi环境下的IP地址
获取本机在WiFi环境下的ip地址，在上一节中其实已经写过，这里将其提取出来即可：
```
- (NSString *)getLocalIPAddressForCurrentWiFi
{
    NSString *address = nil;
    struct ifaddrs *interfaces = NULL;
    struct ifaddrs *temp_addr = NULL;
    int success = 0;

    // retrieve the current interfaces - returns 0 on success
    success = getifaddrs(&interfaces);
    if (success == 0) {
        // Loop through linked list of interfaces
        temp_addr = interfaces;
        while(temp_addr != NULL) {
            if(temp_addr->ifa_addr->sa_family == AF_INET) {
                // Check if interface is en0 which is the wifi connection on the iPhone
                if([[NSString stringWithUTF8String:temp_addr->ifa_name] isEqualToString:@"en0"]) {
                    address = [NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_addr)->sin_addr)];
                    return address;
                }
            }
            temp_addr = temp_addr->ifa_next;
        }
        freeifaddrs(interfaces);
    }
    return nil;
}
```
同样的方式也可以获取广播地址、子网掩码、端口等，组装成一个字典。
```
- (NSMutableDictionary *)getLocalInfoForCurrentWiFi {

    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    struct ifaddrs *interfaces = NULL;
    struct ifaddrs *temp_addr = NULL;
    int success = 0;

    // retrieve the current interfaces - returns 0 on success
    success = getifaddrs(&interfaces);
    if (success == 0) {
        // Loop through linked list of interfaces
        temp_addr = interfaces;
        //*/
        while(temp_addr != NULL) {
            if(temp_addr->ifa_addr->sa_family == AF_INET) {
                // Check if interface is en0 which is the wifi connection on the iPhone
                if([[NSString stringWithUTF8String:temp_addr->ifa_name] isEqualToString:@"en0"]) {
                    //----192.168.1.255 广播地址
                    NSString *broadcast = [NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_dstaddr)->sin_addr)];
                    if (broadcast) {
                        [dict setObject:broadcast forKey:@"broadcast"];
                    }
                    NSLog(@"broadcast address--%@",broadcast);
                    //--192.168.1.106 本机地址
                    NSString *localIp = [NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_addr)->sin_addr)];
                    if (localIp) {
                        [dict setObject:localIp forKey:@"localIp"];
                    }
                    NSLog(@"local device ip--%@",localIp);
                    //--255.255.255.0 子网掩码地址
                    NSString *netmask = [NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_netmask)->sin_addr)];
                    if (netmask) {
                        [dict setObject:netmask forKey:@"netmask"];
                    }
                    NSLog(@"netmask--%@",netmask);
                    //--en0 端口地址
                    NSString *interface = [NSString stringWithUTF8String:temp_addr->ifa_name];
                    if (interface) {
                        [dict setObject:interface forKey:@"interface"];
                    }
                    NSLog(@"interface--%@",interface);
                    return dict;
                }
            }

            temp_addr = temp_addr->ifa_next;
        }
    }

    // Free memory
    freeifaddrs(interfaces);
    return dict;
}
```
将返回的字典打印出来：
```
2016-05-12 17:59:09.257 SimplePing[19141:6830567] wifi:{
    broadcast = "192.168.1.255";
    interface = en0;
    localIp = "192.168.1.7";
    netmask = "255.255.255.0";
}
```

