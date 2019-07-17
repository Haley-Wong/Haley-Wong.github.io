---
layout:     post
title:      "iOS CoreBluetooth 的使用讲解"
date:       2016-04-26
author:     "Haley_Wong"
catalog:    true
tags:
    - Bluetooth
---

最近研究了iOS下连接蓝牙打印机，实现打印购物小票的功能，对iOS中BLE 4.0的使用有了一定的了解，这里记录一下对BLE 4.0的理解。

由于很多文章同时讲CBCentralManager和CBPeripheralManager，所以很容易傻傻分不清楚。很少把iPhone作为蓝牙外设在广播发送数据的情形，今天我就从iOS app开发的角度讲一些BLE 4.0的使用。

# 概念
`CBPeripheral` 蓝牙外设，比如蓝牙手环、蓝牙心跳监视器、蓝牙打印机。

`CBCentralManager` 蓝牙外设管理中心，与手机的蓝牙硬件模板关联，可以获取到手机中蓝牙模块的一些状态等，但是管理的就是蓝牙外设。

`CBService` 蓝牙外设的服务，每一个蓝牙外设都有0个或者多个服务。而每一个蓝牙服务又可能包含0个或者多个蓝牙服务，也可能包含0个或者多个蓝牙特性。

`CBCharacteristic` 每一个蓝牙特性中都包含有一些数据或者信息。

![BLE之间的关系图.png](/img/blogs/bluetooth-summary/img_01.png)

# 分析

我们一般的交互，是app作为客户端，而用户的实际数据多存储在服务器上，所以app客户端主动通过网络接口从服务器端获取数据，然后在app中展示这些数据。

而蓝牙有一些不同，app是外设管理中心(CBCentralManager)，但是它也是客户端。而实际的数据是从蓝牙外设(CBPeripheral)，也就是蓝牙手环等这类设备中获取，所以CBPeripheral就相当于是服务器，与他们有些不同的是，蓝牙数据传输是服务器(CBPeripheral)一直在广播发送数据，app客户端连接监听某个蓝牙后，就会收到其发送过来的数据展示。

蓝牙外设，不管有没有别的设备连接它，蓝牙外设都会广播发送数据。

**情景一 只涉及从蓝牙外设中读数据**

*蓝牙手环*

蓝牙手环一直往外广播发送心跳和走路的步数，当我们的app通过蓝牙连接到蓝牙手环后，就可以在外设的代理方法中，获取广播发出的数据了，然后在app的UI中更新数据即可。

**情景二 往蓝牙外设中写数据**

*蓝牙打印机*

蓝牙打印机是app中通过蓝牙连接到蓝牙打印机之后，利用外设的代理方法，往蓝牙打印机中写入数据后，蓝牙打印机就会自动打印出小票。

**情景三 两台iOS 设备通过app互传文件**

一台设备不能既是外设，又是管理中心。

它可以既广播发送数据，又获取其他设备的数据，但是它只能扮演一种角色，如果iOS 设备A 通过蓝牙主动连接了 设备B，那么设备A是`CBCentral`，设备B是`CBPeripheral`；但是如果是设备B连接了设备A，那么设备B就是`CBCentral`,设备A是`CBPeripheral`。

# 代码实战
>  * 第一步，创建CBCentralManager。
>  * 第二步，扫描可连接的蓝牙外设（必须在蓝牙模块打开的前提下）。
>  * 第三步，连接目标蓝牙外设。
>  * 第四步，查询目标蓝牙外设下的服务。
>  * 第五步，遍历服务中的特性，获取特性中的数据或者保存某些可写的特性，或者设置某些特性值改变时，通知主动获取。
>  * 第六步，在通知更新特性中值的方法中读取特性中的数据（再设置特性的通知为YES的情况下）。
>  * 第七步，读取特性中的值。
>  * 第八步，如果有可写特性，并且需要向蓝牙外设写入数据时，写入数据发送给蓝牙外设。

首先是是在我们app中，创建一个`CBCentralManager`:
```
// 1.创建管理中心，这里也可以设置子线程
    CBCentralManager *manager = [[CBCentralManager alloc] initWithDelegate:self queue:dispatch_get_main_queue()];
```

创建完之后，就会调用一次`CBCentralManagerDelegate`的代理方法：

```
- (void)centralManagerDidUpdateState:(CBCentralManager *)central
{
    NSLog(@"%@",central);
    switch (central.state) {
        case CBCentralManagerStatePoweredOn:
            NSLog(@"打开，可用");
            [_manager scanForPeripheralsWithServices:nil options:@{CBCentralManagerScanOptionAllowDuplicatesKey:@(NO)}];
            break;
        case CBCentralManagerStatePoweredOff:
            NSLog(@"可用，未打开");
            break;
        case CBCentralManagerStateUnsupported:
            NSLog(@"SDK不支持");
            break;
        case CBCentralManagerStateUnauthorized:
            NSLog(@"程序未授权");
            break;
        case CBCentralManagerStateResetting:
            NSLog(@"CBCentralManagerStateResetting");
            break;
        case CBCentralManagerStateUnknown:
            NSLog(@"CBCentralManagerStateUnknown");
            break;
    }
}
```
该代理方法，在蓝牙模板的状态发生改变的时候，就会回调。应该在蓝牙打开的状态下，再去搜索扫描可用的蓝牙外设列表。

扫描蓝牙外设是通过如下方法：
```
- (void)scanForPeripheralsWithServices:(nullable NSArray<CBUUID *> *)serviceUUIDs options:(nullable NSDictionary<NSString *, id> *)options;
```

第一个参数是服务的CBUUID数组，我们可以搜索具有某一类服务的蓝牙设备，比较重要。
扫描到蓝牙外设后，会调用`CBCentralManagerDelegate`的这个代理方法：
```
- (void)centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)peripheral advertisementData:(NSDictionary<NSString *, id> *)advertisementData RSSI:(NSNumber *)RSSI;
```
该方法一次只返回一个蓝牙外设的信息。第二个参数是扫描到的蓝牙外设，第三个参数是蓝牙外设中
的额外数据，RSSI是信号强度的参数。

因为可能某个蓝牙是无用的或者重复扫描到某一个蓝牙，所以我们需要剔除一些无用的蓝牙，替换掉旧的蓝牙外设（可能该外设的参数有变化，不是携带的数据，是外设本身的参数变化）。

```
- (void)centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)peripheral advertisementData:(NSDictionary<NSString *, id> *)advertisementData RSSI:(NSNumber *)RSSI
{
    if (peripheral.name.length <= 0) {
        return ;
    }
    
    NSLog(@"Discovered name:%@,identifier:%@,advertisementData:%@,RSSI:%@", peripheral.name, peripheral.identifier,advertisementData,RSSI);
    if (self.deviceArray.count == 0) {
        NSDictionary *dict = @{@"peripheral":peripheral, @"RSSI":RSSI};
        [self.deviceArray addObject:dict];
    } else {
        BOOL isExist = NO;
        for (int i = 0; i < self.deviceArray.count; i++) {
            NSDictionary *dict = [self.deviceArray objectAtIndex:i];
            CBPeripheral *per = dict[@"peripheral"];
            if ([per.identifier.UUIDString isEqualToString:peripheral.identifier.UUIDString]) {
                isExist = YES;
                NSDictionary *dict = @{@"peripheral":peripheral, @"RSSI":RSSI};
                [_deviceArray replaceObjectAtIndex:i withObject:dict];
            }
        }
        
        if (!isExist) {
            NSDictionary *dict = @{@"peripheral":peripheral, @"RSSI":RSSI};
            [self.deviceArray addObject:dict];
        }
    }
    
    [self.tableView reloadData];
}
```
这样就获取到了蓝牙设备列表，我们可以在表格中展示蓝牙设备列表

![蓝牙外设列表.png](/img/blogs/bluetooth-summary/img_02.png)

到这里只获取到了可连接的蓝牙外设，当我们连接到某个蓝牙外设后，就可以去获取它的数据了。

在cell点击事件中连接某个蓝牙外设：
```
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
    NSDictionary *dict = [self.deviceArray objectAtIndex:indexPath.row];
    CBPeripheral *peripheral = dict[@"peripheral"];
    // 连接某个蓝牙外设
    [self.manager connectPeripheral:peripheral options:@{CBConnectPeripheralOptionNotifyOnDisconnectionKey:@(YES)}];
    // 设置外设的代理是为了后面查询外设的服务和外设的特性，以及特性中的数据。
    [peripheral setDelegate:self];
    // 既然已经连接到某个蓝牙了，那就不需要在继续扫描外设了
    [self.manager stopScan];
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
}
```
连接某个外设成功后，查找其具有的服务
```
- (void)centralManager:(CBCentralManager *)central didConnectPeripheral:(CBPeripheral *)peripheral
{
    NSLog(@"didConnectPeripheral");
    // 连接成功后，查找服务
    [peripheral discoverServices:nil];
}

- (void)centralManager:(CBCentralManager *)central didFailToConnectPeripheral:(CBPeripheral *)peripheral error:(nullable NSError *)error
{
    NSLog(@"didFailToConnectPeripheral");
}
```
查找服务的代理方法就是`CBPeripheralDelegate`中的了：
```
#pragma mark - CBPeripheralDelegate
- (void)peripheral:(CBPeripheral *)peripheral didDiscoverServices:(nullable NSError *)error
{
    NSString *UUID = [peripheral.identifier UUIDString];
    NSLog(@"didDiscoverServices:%@",UUID);
    if (error) {
        NSLog(@"出错");
        return;
    }
    
    CBUUID *cbUUID = [CBUUID UUIDWithString:UUID];
    NSLog(@"cbUUID:%@",cbUUID);
    
    for (CBService *service in peripheral.services) {
        NSLog(@"service:%@",service.UUID);
        //如果我们知道要查询的特性的CBUUID，可以在参数一中传入CBUUID数组。
        [peripheral discoverCharacteristics:nil forService:service];
    }
}
```
再然后是遍历服务中的特性：
```
- (void)peripheral:(CBPeripheral *)peripheral didDiscoverCharacteristicsForService:(CBService *)service error:(nullable NSError *)error
{
    if (error) {
        NSLog(@"出错");
        return;
    }
    
    for (CBCharacteristic *character in service.characteristics) {
        // 这是一个枚举类型的属性
        CBCharacteristicProperties properties = character.properties;
        if (properties & CBCharacteristicPropertyBroadcast) {
            //如果是广播特性
        }
        
        if (properties & CBCharacteristicPropertyRead) {
            //如果具备读特性，即可以读取特性的value
            [peripheral readValueForCharacteristic:character];
        }
        
        if (properties & CBCharacteristicPropertyWriteWithoutResponse) {
            //如果具备写入值不需要响应的特性
            //这里保存这个可以写的特性，便于后面往这个特性中写数据
            _chatacter = character;
        }
        
        if (properties & CBCharacteristicPropertyWrite) {
            //如果具备写入值的特性，这个应该会有一些响应
        }
        
        if (properties & CBCharacteristicPropertyNotify) {
            //如果具备通知的特性，无响应
            [peripheral setNotifyValue:YES forCharacteristic:character];
        }
    }
}
```
然后通知的代理方法如下：
```
- (void)peripheral:(CBPeripheral *)peripheral didUpdateNotificationStateForCharacteristic:(nonnull CBCharacteristic *)characteristic error:(nullable NSError *)error
{
    if (error) {
        NSLog(@"错误didUpdateNotification：%@",error);
        return;
    }
    
    CBCharacteristicProperties properties = characteristic.properties;
    if (properties & CBCharacteristicPropertyRead) {
        //如果具备读特性，即可以读取特性的value
        [peripheral readValueForCharacteristic:characteristic];
    }
}
```

读取特性中的value的方法如下：
```
// 读取新值的结果
- (void)peripheral:(CBPeripheral *)peripheral didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
{
    if (error) {
        NSLog(@"错误：%@",error);
        return;
    }
    
    NSData *data = characteristic.value;
    if (data.length <= 0) {
        return;
    }
    NSString *info = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    
    NSLog(@"info:%@",info);
}
```
到这里，获取蓝牙外设广播发送出来的值就已经完毕了。

想要向蓝牙外设写入数据，则调用如下方法：
```
 [peripheral writeValue:infoData forCharacteristic:_chatacter type:CBCharacteristicWriteWithoutResponse];
```
只是这里的_chatacter参数应该是遍历服务器的特性时，遍历出来的那个可写的特性。如果蓝牙外设没有可写特性，则不能向其写入数据。

另外取消与某蓝牙外设的连接方法是：
```
[self.manager cancelPeripheralConnection:peripheral];
```
`CBCentralManagerDelegate`中也有断开蓝牙连接的代理方法：
```
- (void)centralManager:(CBCentralManager *)central didDisconnectPeripheral:(CBPeripheral *)peripheral error:(nullable NSError *)error;
```

## iOS 10 补充
经 [@一脚踢飞](http://www.jianshu.com/users/7580a99c35df)提醒：[https://developer.apple.com/reference/corebluetooth?language=objc](https://developer.apple.com/reference/corebluetooth?language=objc)
> 
**Important:** To protect user privacy, an iOS app linked on or after iOS 10.0, and which accesses the Bluetooth interface, must statically declare the intent to do so. Include the NSBluetoothPeripheralUsageDescription key in your app’s Info.plist
 file and provide a purpose string for this key. If your app attempts to access the Bluetooth interface without a corresponding purpose string, your app exits。
**tip:** This key is supported in iOS 6.0 and later.

但是我测试在iOS 10.0.1中测试，不加`NSBluetoothPeripheralUsageDescription`，工程仍然可以正常使用。

然后加上`NSBluetoothPeripheralUsageDescription`后，

![](/img/blogs/bluetooth-summary/img_03.png)

应用启动时也并没有像定位、推送等那样的提示😞 😞 😞。在设置中，蓝牙功能目前还并未看到允许使用的应用列表，估计苹果只是在未来规划的吧。

## 补充
鉴于经常有人问为啥工程里能搜到蓝牙打印机，但是却搜不到其他手机的蓝牙？

那是因为蓝牙技术发展至今，也从 1.x 发展到 4.0了，蓝牙通信使用的材料、技术等都发生了变化。这就是为什么有的打印机支持 2.0、3.0、4.0，如果你使用的是CoreBluetooth库，而打印机不支持 蓝牙 4.0，那你当然搜索不到蓝牙打印机啦！ 手机设置里的蓝牙搜索功能，使用的是什么技术实现的，有木有兼容 2.0、3.0、4.0那就不得而知了。

而 iOS 中的 蓝牙库 也不止 `CoreBluetooth` 一个，还有其他的呢！

`GameKit.framework`：iOS7之前的蓝牙通讯框架，从iOS7开始过期，但是目前多数应用还是基于此框架。

`MultipeerConnectivity.framework`：iOS7开始引入的新的蓝牙通讯开发框架，用于取代GameKit。

`CoreBluetooth.framework`：功能强大的蓝牙开发框架，要求设备必须支持蓝牙4.0。

更多关于蓝牙相关的知识：

[蓝牙--百度百科](http://baike.baidu.com/link?url=RrMHcSluLX38YIfOiNCDXVLllFpvVf_aryl0ocI0OQ_jVnP9O_feY7rc3LKn1l1kdok9qiu4hfz_oJUYDjjfaOaCFhiFso-IsdxfQV0ilGO)

[可以只看iOS中三个蓝牙库的介绍](http://www.cnblogs.com/kenshincui/p/4220402.html#bluetooth)

到这里蓝牙的基本使用就结束了！ Have fun！

