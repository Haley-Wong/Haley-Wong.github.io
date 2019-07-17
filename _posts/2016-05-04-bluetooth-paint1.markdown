---
layout:     post
title:      "iOS Bluetooth 打印小票(一)"
date:       2016-05-04
author:     "Haley_Wong"
catalog:    true
tags:
    - Bluetooth
---

在iOS app中连接蓝牙打印机打印商品小票，在没有电脑只有手机的情况下，感觉非常实用，而且最近经常最近公司正好也在做这个功能，所以就研究了下。这一篇主要讲一下打印机的一些命令，以便下一篇文章中使用。

# 蓝牙打印机命令
在蓝牙打印中，比较麻烦的不是搜索，连接蓝牙设备，而是小票的排版。而要弄出好看的小票排版，就得先熟知蓝牙打印机的各种命令。我是在demo基本完成之后，才找到了详细的命令表，如果我先搜索到这份较详细的命令的话，肯定会节省不少时间。现在写出来，希望能帮助其他在做这个功能的人。
`其实每个品牌的打印机，在官网的下载里都有完整的打印机指令文档，记得去下载哦。`

打印机分了很多型号，不同的打印机所使用的指令集可能不同，比如Star打印机和Epson打印机，他们的所使用的指令集就不太一样。这里有篇文章，有几个常用的指令对比: [这是地址](http://www.360doc.com/content/11/1101/10/8005503_160738911.shtml)

我就只记录一种命令集：ESC/POS打印命令集。而一般的打印机支持三种格式：ASCII、十进制、十六进制。

这里有一份PDF文件说明了各个命令的作用和对应的三种格式：[地址](http://www.xmjjdz.com/downloads/manual/cn/ESC(POS)%E6%89%93%E5%8D%B0%E6%8E%A7%E5%88%B6%E5%91%BD%E4%BB%A4.pdf)
# 打印命令一览表
表中都是用ASCII码格式，不要急，下面会介绍每一个命令的十进制和十六进制格式和说明。

![一览表.png](/img/blogs/bluetooth-paint1/img_01.png)
# 打印的各个命令详解
等会每个命令会按照如下格式贴出：
![说明.png](/img/blogs/bluetooth-paint1/img_02.png)
## 1.初始化命令

![初始化命令.png](/img/blogs/bluetooth-paint1/img_03.png)

## 2.打印命令
打印命令有两种：
![打印命令.png](/img/blogs/bluetooth-paint1/img_04.png)
## 3.行间距设置命令
![行间距设置命令.png](/img/blogs/bluetooth-paint1/img_05.png)
## 4.对齐方式设置
![对齐方式设置.png](/img/blogs/bluetooth-paint1/img_06.png)
> 说明：
对齐方式有两种，对应的十六进制 {0x1B,0x61,0x00}、{0x1B,0x61,0x01}、{0x1B,0x61,0x02}
或者 {0x1B,0x61,0x30}、{0x1B,0x61,0x31}、{0x1B,0x61,0x32}。

## 5.字符设置命令
![字符设置命令1.png](/img/blogs/bluetooth-paint1/img_07.png)

![字符设置命令2.png](/img/blogs/bluetooth-paint1/img_08.png)

![字符设置命令3.png](/img/blogs/bluetooth-paint1/img_09.png)

![字符设置命令4.png](/img/blogs/bluetooth-paint1/img_10.png)

## 6.钱箱控制命令

![钱箱控制命令.png](/img/blogs/bluetooth-paint1/img_11.png)

## 7.按键控制命令

![按键控制命令.png](/img/blogs/bluetooth-paint1/img_12.webp)

## 8.图形打印命令

![设定点图命令.png](/img/blogs/bluetooth-paint1/img_13.png)

![打印下装点图.png](/img/blogs/bluetooth-paint1/img_14.png)


## 9.状态传输命令

![向主机传送打印机状态.png](/img/blogs/bluetooth-paint1/img_15.png)

![状态传输命令.png](/img/blogs/bluetooth-paint1/img_16.png)

## 10.条码打印命令

![条码命令](/img/blogs/bluetooth-paint1/img_17.png)


![条码打印](/img/blogs/bluetooth-paint1/img_18.png)

## 11.位置和页模式命令

![位置和页模式命令1](/img/blogs/bluetooth-paint1/img_19.png)

![位置和页模式命令2](/img/blogs/bluetooth-paint1/img_20.png)

![位置和页模式命令3](/img/blogs/bluetooth-paint1/img_21.png)

## 12.切纸模式命令

![切纸模式命令](/img/blogs/bluetooth-paint1/img_22.png)

以上是我找到的比较完整的命令集合说明，希望能帮到他人。

# 其他
这里有其他简友`伊布林`提供的另一份打印机指令集的文档地址：
[打印机指令集文档](http://wenku.baidu.com/link?url=WtNOI1ojokVsXT3LiWmCRg6IAyNRm5U_rwq_-Os_C_aDnZBtjtTNMKTANv-RpyP_7fgRJLCc56KCyDNky6-7x2tAIFEg0aX3D42jSzcIJqW)

这里有我最初用最原始的指令集拼接出来的NSData代码片段，供大家参考：
```
// 打印机支持的文字编码
            NSLog(@"goodsArray:%@",goodsArray);
// 用到的goodsArray跟github中的商品数组是一样的。
            NSStringEncoding enc = CFStringConvertEncodingToNSStringEncoding(kCFStringEncodingGB_18030_2000);
            
            NSString *title = @"测试电商";
            NSString *str1 = @"测试电商服务中心(销售单)";
            NSString *line = @"- - - - - - - - - - - - - - - -";
            NSString *time = @"时间:2016-04-27 10:01:50";
            NSString *orderNum = @"订单编号:4000020160427100150";
            NSString *address = @"地址:深圳市南山区学府路东科技园店";
            
            //初始化打印机
            Byte initBytes[] = {0x1B,0x40};
            NSData *initData = [NSData dataWithBytes:initBytes length:sizeof(initBytes)];
            
            //换行
            Byte nextRowBytes[] = {0x0A};
            NSData *nextRowData = [NSData dataWithBytes:nextRowBytes length:sizeof(nextRowBytes)];
            
            //居中
            Byte centerBytes[] = {0x1B,0x61,1};
            NSData *centerData= [NSData dataWithBytes:centerBytes length:sizeof(centerBytes)];
            
            //居左
            Byte leftBytes[] = {0x1B,0x61,0};
            NSData *leftdata= [NSData dataWithBytes:leftBytes length:sizeof(leftBytes)];
            
            NSMutableData *mainData = [[NSMutableData alloc]init];
            
            //初始化打印机
            [mainData appendData:initData];
            //设置文字居中/居左
            [mainData appendData:centerData];
            [mainData appendData:[title dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            [mainData appendData:[str1 dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            
//            UIImage *qrImage =[MMQRCode createBarImageWithOrderStr:@"RN3456789012"];
//            UIImage *qrImage =[MMQRCode qrCodeWithString:@"http://www.sina.com" logoName:nil size:400];
//            qrImage = [self scaleCurrentImage:qrImage];
//            
//            NSData *data = [IGThermalSupport imageToThermalData:qrImage];
//            [mainData appendData:centerData];
//            [mainData appendData:data];
//            [mainData appendData:nextRowData];
            
            [mainData appendData:leftdata];
            [mainData appendData:[line dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            [mainData appendData:[time dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            [mainData appendData:[orderNum dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            [mainData appendData:[address dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            
            [mainData appendData:[line dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            NSString *name = @"商品";
            NSString *number = @"数量";
            NSString *price = @"单价";
            [mainData appendData:leftdata];
            [mainData appendData:[name dataUsingEncoding:enc]];
            
            Byte spaceBytes1[] = {0x1B, 0x24, 150 % 256, 0};
            NSData *spaceData1 = [NSData dataWithBytes:spaceBytes1 length:sizeof(spaceBytes1)];
            [mainData appendData:spaceData1];
            [mainData appendData:[number dataUsingEncoding:enc]];
            
            Byte spaceBytes2[] = {0x1B, 0x24, 300 % 256, 1};
            NSData *spaceData2 = [NSData dataWithBytes:spaceBytes2 length:sizeof(spaceBytes2)];
            [mainData appendData:spaceData2];
            [mainData appendData:[price dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            
            CGFloat total = 0.0;
            for (NSDictionary *dict in goodsArray) {
                [mainData appendData:[dict[@"name"] dataUsingEncoding:enc]];
                
                Byte spaceBytes1[] = {0x1B, 0x24, 150 % 256, 0};
                NSData *spaceData1 = [NSData dataWithBytes:spaceBytes1 length:sizeof(spaceBytes1)];
                [mainData appendData:spaceData1];
                [mainData appendData:[dict[@"amount"] dataUsingEncoding:enc]];
                
                Byte spaceBytes2[] = {0x1B, 0x24, 300 % 256, 1};
                NSData *spaceData2 = [NSData dataWithBytes:spaceBytes2 length:sizeof(spaceBytes2)];
                [mainData appendData:spaceData2];
                [mainData appendData:[dict[@"price"] dataUsingEncoding:enc]];
                [mainData appendData:nextRowData];
                
                total += [dict[@"price"] floatValue] * [dict[@"amount"] intValue];
            }
            
            [mainData appendData:[line dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            [mainData appendData:[@"总计:" dataUsingEncoding:enc]];
            Byte spaceBytes[] = {0x1B, 0x24, 300 % 256, 1};
            NSData *spaceData = [NSData dataWithBytes:spaceBytes length:sizeof(spaceBytes)];
            [mainData appendData:spaceData];
            NSString *totalStr = [NSString stringWithFormat:@"%.2f",total];
            [mainData appendData:[totalStr dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            
            [mainData appendData:[line dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            [mainData appendData:centerData];
            [mainData appendData:[@"谢谢惠顾，欢迎下次光临!" dataUsingEncoding:enc]];
            [mainData appendData:nextRowData];
            
            
            [self.peripheral writeValue:mainData forCharacteristic:self.chatacter type:CBCharacteristicWriteWithoutResponse];
```

#  打印没反应？

如果你连接成功，但是发出打印指令后，打印机没反应，很有可能是因为你的打印机一次发送的数据长度小于146，你把146改的更小一点试试看。

我测试的两台佳博打印机，一台没有长度限制，一台最多每次只能发送146个字节，否则会出现打印没反应的情况，需要重启打印机。

不同的打印机，可能对长度的限制不太一样，据群友反应有的打印机只能支持一次发送20个字节，所以你需要将宏里面的146改的更小一点。

Have Fun!

