---
layout:     post
title:      "iOS Bluetooth 打印小票(二)"
date:       2016-05-04
author:     "Haley_Wong"
catalog:    true
tags:
    - Bluetooth
---

在上一篇中介绍了打印小票所需要的命令，这一篇介绍Bluetooth连接蓝牙和打印小票的全过程。

![小票](/img/blogs/bluetooth-paint2/img_01.webp)

# CoreBluetooth的封装
因为CoreBluetooth中的代理太多，而每一次操作又比较依赖上一次操作的结果，方法又比较零散，所以我做了粗略封装，把代理改成了block方式回调。

## 1.获取蓝牙管理单例
```
HLBLEManager *manager = [HLBLEManager sharedInstance];
    __weak HLBLEManager *weakManager = manager;
    manager.stateUpdateBlock = ^(CBCentralManager *central) {
        NSString *info = nil;
        switch (central.state) {
            case CBCentralManagerStatePoweredOn:
                info = @"蓝牙已打开，并且可用";
                //三种种方式
                // 方式1
                [weakManager scanForPeripheralsWithServiceUUIDs:nil options:nil];
                // 方式2
                [central scanForPeripheralsWithServices:nil options:nil];
                // 方式3
                [weakManager scanForPeripheralsWithServiceUUIDs:nil options:nil didDiscoverPeripheral:^(CBCentralManager *central, CBPeripheral *peripheral, NSDictionary *advertisementData, NSNumber *RSSI) {
                    
                }];
                break;
            case CBCentralManagerStatePoweredOff:
                info = @"蓝牙可用，未打开";
                break;
            case CBCentralManagerStateUnsupported:
                info = @"SDK不支持";
                break;
            case CBCentralManagerStateUnauthorized:
                info = @"程序未授权";
                break;
            case CBCentralManagerStateResetting:
                info = @"CBCentralManagerStateResetting";
                break;
            case CBCentralManagerStateUnknown:
                info = @"CBCentralManagerStateUnknown";
                break;
        }
        
        [SVProgressHUD setDefaultStyle:SVProgressHUDStyleDark];
        [SVProgressHUD showInfoWithStatus:info ];
    };
```
因为CBCentralManager一创建，就会在代理中返回蓝牙模块的状态，所以及时设置状态返回的回调，以便在搜索附近可用的蓝牙外设。

## 2.搜索可用的蓝牙外设
```
                // 方式1
                [weakManager scanForPeripheralsWithServiceUUIDs:nil options:nil];
                // 方式2
                [central scanForPeripheralsWithServices:nil options:nil];
                // 方式3
                [weakManager scanForPeripheralsWithServiceUUIDs:nil options:nil didDiscoverPeripheral:^(CBCentralManager *central, CBPeripheral *peripheral, NSDictionary *advertisementData, NSNumber *RSSI) {
                    
                }];
```
这里给出了三种方式，前两种方式都需要先设置好搜索到蓝牙外设之后的回调，即：
```
    manager.discoverPeripheralBlcok = ^(CBCentralManager *central, CBPeripheral *peripheral, NSDictionary *advertisementData, NSNumber *RSSI) {
        if (peripheral.name.length <= 0) {
            return ;
        }
        
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
        
    };
}
```

第三种方式，则附带一个block，便于直接处理。

## 3.连接蓝牙外设
```
HLBLEManager *manager = [HLBLEManager sharedInstance];
    [manager connectPeripheral:_perpheral
                connectOptions:@{CBConnectPeripheralOptionNotifyOnDisconnectionKey:@(YES)}
        stopScanAfterConnected:YES
               servicesOptions:nil
        characteristicsOptions:nil
                 completeBlock:^(HLOptionStage stage, CBPeripheral *peripheral, CBService *service, CBCharacteristic *character, NSError *error) {
                     switch (stage) {
                         case HLOptionStageConnection:
                         {
                             if (error) {
                                 [SVProgressHUD showErrorWithStatus:@"连接失败"];
                                 
                             } else {
                                 [SVProgressHUD showSuccessWithStatus:@"连接成功"];
                             }
                             break;
                         }
                         case HLOptionStageSeekServices:
                         {
                             if (error) {
                                 [SVProgressHUD showSuccessWithStatus:@"查找服务失败"];
                             } else {
                                 [SVProgressHUD showSuccessWithStatus:@"查找服务成功"];
                                 [_infos addObjectsFromArray:peripheral.services];
                                 [_tableView reloadData];
                             }
                             break;
                         }
                         case HLOptionStageSeekCharacteristics:
                         {
                             // 该block会返回多次，每一个服务返回一次
                             if (error) {
                                 NSLog(@"查找特性失败");
                             } else {
                                 NSLog(@"查找特性成功");
                                 [_tableView reloadData];
                             }
                             break;
                         }
                         case HLOptionStageSeekdescriptors:
                         {
                             // 该block会返回多次，每一个特性返回一次
                             if (error) {
                                 NSLog(@"查找特性的描述失败");
                             } else {
                                 NSLog(@"查找特性的描述成功");
                             }
                             break;
                         }
                         default:
                             break;
                     }
                     
                 }];
```
因为连接蓝牙外设--->扫描蓝牙外设服务--->扫描蓝牙外设服务特性--->扫描特性描述
这些操作都是有阶段性的，并且依赖上一步的结果。
这里我也给出了两种方式：
方式一(推荐)：如上面代码一样，设置最后一个参数block,然后在block中判断当前是哪个阶段的回调。
方式二：提前设置好每一阶段的block，然后设置方法中最后一个参数的block为nil
```
/** 连接外设完成的回调 */
@property (copy, nonatomic) HLConnectCompletionBlock                connectCompleteBlock;
/** 发现服务的回调 */
@property (copy, nonatomic) HLDiscoveredServicesBlock               discoverServicesBlock;
/** 发现服务中的特性的回调 */
@property (copy, nonatomic) HLDiscoverCharacteristicsBlock          discoverCharacteristicsBlock;
/** 发现特性的描述的回调 */
@property (copy, nonatomic) HLDiscoverDescriptorsBlock              discoverDescriptorsBlock;
```
## 4.记录下蓝牙外设中的可写特性
记录下特性中的可写服务以便，往这个蓝牙外设中写入数据。
```
CBCharacteristic *character = [service.characteristics objectAtIndex:indexPath.row];
    CBCharacteristicProperties properties = character.properties;
    if (properties & CBCharacteristicPropertyWriteWithoutResponse) {
        self.chatacter = character;
    }
```
## 5.拼装要写入到蓝牙的数据
```
        NSString *title = @"测试电商";
        NSString *str1 = @"测试电商服务中心(销售单)";
        
        HLPrinter *printer = [[HLPrinter alloc] init];
        [printer appendText:title alignment:HLTextAlignmentCenter fontSize:HLFontSizeTitleBig];
        [printer appendText:str1 alignment:HLTextAlignmentCenter];
        [printer appendBarCodeWithInfo:@"RN3456789012"];
        [printer appendSeperatorLine];
        
        [printer appendTitle:@"时间:" value:@"2016-04-27 10:01:50" valueOffset:150];
        [printer appendTitle:@"订单:" value:@"4000020160427100150" valueOffset:150];
        [printer appendText:@"地址:深圳市南山区学府路东深大店" alignment:HLTextAlignmentLeft];
        
        [printer appendSeperatorLine];
        [printer appendLeftText:@"商品" middleText:@"数量" rightText:@"单价" isTitle:YES];
        CGFloat total = 0.0;
        for (NSDictionary *dict in goodsArray) {
            [printer appendLeftText:dict[@"name"] middleText:dict[@"amount"] rightText:dict[@"price"] isTitle:NO];
            total += [dict[@"price"] floatValue] * [dict[@"amount"] intValue];
        }
        
        [printer appendSeperatorLine];
        NSString *totalStr = [NSString stringWithFormat:@"%.2f",total];
        [printer appendTitle:@"总计:" value:totalStr];
        [printer appendTitle:@"实收:" value:@"100.00"];
        NSString *leftStr = [NSString stringWithFormat:@"%.2f",100.00 - total];
        [printer appendTitle:@"找零:" value:leftStr];
        
        [printer appendFooter:nil];
        
        [printer appendImage:[UIImage imageNamed:@"ico180"] alignment:HLTextAlignmentCenter maxWidth:300];
        
        NSData *mainData = [printer getFinalData];
```
## 6.写入数据
```
HLBLEManager *bleManager = [HLBLEManager sharedInstance];
        [bleManager writeValue:mainData forCharacteristic:self.chatacter type:CBCharacteristicWriteWithoutResponse];
```
写入数据后，蓝牙打印机就会开始打印小票。

# 蓝牙打印机操作封装
## 1.创建一个打印操作对象
```
HLPrinter *printer = [[HLPrinter alloc] init];
```
在创建这个打印机操作对象时，内部做了很多预设置：
```
- (instancetype)init
{
    self = [super init];
    if (self) {
        [self defaultSetting];
    }
    return self;
}

- (void)defaultSetting
{
    _printerData = [[NSMutableData alloc] init];
    
    // 1.初始化打印机
    Byte initBytes[] = {0x1B,0x40};
    [_printerData appendBytes:initBytes length:sizeof(initBytes)];
    // 2.设置行间距为1/6英寸，约34个点
    // 另一种设置行间距的方法看这个 @link{-setLineSpace:}
    Byte lineSpace[] = {0x1B,0x32};
    [_printerData appendBytes:lineSpace length:sizeof(lineSpace)];
    // 3.设置字体:标准0x00，压缩0x01;
    Byte fontBytes[] = {0x1B,0x4D,0x00};
    [_printerData appendBytes:fontBytes length:sizeof(fontBytes)];
}
```
## 2.设置要打印的内容

可以打印的内容包括：文字、二维码、条形码、图片。
而对这些内容的处理已经做了封装，只需要简单调用某些API即可。

### 2.1 打印单行文字
```
/**
 *  添加单行标题,默认字号是小号字体
 *
 *  @param title     标题名称
 *  @param alignment 标题对齐方式
 */
- (void)appendText:(NSString *)text alignment:(HLTextAlignment)alignment;

/**
 *  添加单行标题
 *
 *  @param title     标题名称
 *  @param alignment 标题对齐方式
 *  @param fontSize  标题字号
 */
- (void)appendText:(NSString *)text alignment:(HLTextAlignment)alignment fontSize:(HLFontSize)fontSize;
```
### 2.2 打印左标题，右内容文字
```
/**
 *  添加单行信息，左边名称(左对齐)，右边实际值（右对齐）。
 *  @param title    名称
 *  @param value    实际值
 *  @param fontSize 字号大小
 *  警告:因字号和字体与iOS中字体不一致，计算出来有误差
 */
- (void)appendTitle:(NSString *)title value:(NSString *)value fontSize:(HLFontSize)fontSize;

/**
 *  设置单行信息，左标题，右实际值
 *
 *  @param title    标题
 *  @param value    实际值
 *  @param offset   实际值偏移量
 */
- (void)appendTitle:(NSString *)title value:(NSString *)value valueOffset:(NSInteger)offset;

/**
 *  设置单行信息，左标题，右实际值
 *
 *  @param title    标题
 *  @param value    实际值
 *  @param offset   实际值偏移量
 *  @param fontSize 字号
 */
- (void)appendTitle:(NSString *)title value:(NSString *)value valueOffset:(NSInteger)offset fontSize:(HLFontSize)fontSize;
```
### 3.三列数据样式
```
/**
 *  添加选购商品信息标题,一般是三列，名称、数量、单价
 *
 *  @param LeftText   左标题
 *  @param middleText 中间标题
 *  @param rightText  右标题
 */
- (void)appendLeftText:(NSString *)left middleText:(NSString *)middle rightText:(NSString *)right isTitle:(BOOL)isTitle;
```
### 4.打印条形码
```
/**
 *  添加条形码图片
 *
 *  @param info 条形码中包含的信息，默认居中显示，最大宽度为300。如果大于300,会等比缩放。
 */
- (void)appendBarCodeWithInfo:(NSString *)info;

/**
 *  添加条形码图片
 *
 *  @param info      条形码中的信息
 *  @param alignment 图片对齐方式
 *  @param maxWidth  图片最大宽度
 */
- (void)appendBarCodeWithInfo:(NSString *)info alignment:(HLTextAlignment)alignment maxWidth:(CGFloat)maxWidth;
```
### 5.打印二维码
```
/**
 *  添加二维码图片
 *
 *  @param info 二维码中的信息
 */
- (void)appendQRCodeWithInfo:(NSString *)info;

/**
 *  添加二维码图片
 *
 *  @param info        二维码中的信息
 *  @param centerImage 二维码中间的图片
 *  @param alignment   对齐方式
 *  @param maxWidth    二维码的最大宽度
 */
- (void)appendQRCodeWithInfo:(NSString *)info centerImage:(UIImage *)centerImage alignment:(HLTextAlignment)alignment maxWidth:(CGFloat )maxWidth;
```
### 6.打印图片
```
/**
 *  添加图片，一般是添加二维码或者条形码
 *
 *  @param image     图片
 *  @param alignment 图片对齐方式
 *  @param maxWidth  图片的最大宽度，如果图片过大，会等比缩放
 */
- (void)appendImage:(UIImage *)image alignment:(HLTextAlignment)alignment maxWidth:(CGFloat)maxWidth;
```
### 7.打印分隔线
```
/**
 *  添加一条分割线，like this:---------------------------
 */
- (void)appendSeperatorLine;
```
### 8.打印footer 
```
/**
 *  添加底部信息
 *
 *  @param footerInfo 不填默认为 谢谢惠顾，欢迎下次光临！
 */
- (void)appendFooter:(NSString *)footerInfo;
```
### 9.获取最终数据
```
/**
 *  获取最终的data
 *
 *  @return 最终的data
 */
- (NSData *)getFinalData;
```
![各种样式展示](/img/blogs/bluetooth-paint2/img_02.webp)
而HLPrinter内部实际有一些私有方法，都是对上一篇内容中打印机命令的封装，作为基础操作
例如：
```
/**
 *  换行
 */
- (void)appendNewLine
{
    Byte nextRowBytes[] = {0x0A};
    [_printerData appendBytes:nextRowBytes length:sizeof(nextRowBytes)];
}

/**
 *  回车
 */
- (void)appendReturn
{
    Byte returnBytes[] = {0x0D};
    [_printerData appendBytes:returnBytes length:sizeof(returnBytes)];
}

/**
 *  设置对齐方式
 *
 *  @param alignment 对齐方式：居左、居中、居右
 */
- (void)setAlignment:(HLTextAlignment)alignment
{
    Byte alignBytes[] = {0x1B,0x61,alignment};
    [_printerData appendBytes:alignBytes length:sizeof(alignBytes)];
}

/**
 *  设置字体大小
 *
 *  @param fontSize 字号
 */
- (void)setFontSize:(HLFontSize)fontSize
{
    Byte fontSizeBytes[] = {0x1D,0x21,fontSize};
    [_printerData appendBytes:fontSizeBytes length:sizeof(fontSizeBytes)];
}
```
在`UIImage+Bitmap`中，主要是对图片操作的两个Category,一个是创建二维码、条形码图片。
另一是将图片转换为点阵图数据。

# 补充
可能对于小票的样式不仅仅局限于封装的几种，有人提到左边二维码图片，右边居中显示一些文字的布局方式，这样用原来的指令集组合的方式就很难实现。

对于一些不太好弄的布局样式，我们可以曲线救国，这里有一些新的场景和解决方案：
1. 可以先在容器视图上实现，然后再截取容器视图，将截取后的图片打印出来就可以啦😃 。
2. 用HTML做出订单布局，然后用UIWebView加载出来后，截取WebView完整内容，再打印出来。

用UIWebView打印的方式，还可以在线修改订单的样式和布局，就是比较浪费墨，没有指令集组合的方式打印出来的清晰。

以下是利用UIWebView，然后获取WebView快照打印出来的小票：

![小票](/img/blogs/bluetooth-paint2/img_03.webp)

获取UIWebView的完整内容截图的方法：
```
/**
 *  获取当前加载的网页的截图
 *  获取当前WebView的size,然后一屏一屏的截图后，再拼接成一张完整的图片
 *
 *  @return
 */
- (UIImage *)imageForWebView
{
    // 1.获取WebView的宽高
    CGSize boundsSize = self.bounds.size;
    CGFloat boundsWidth = boundsSize.width;
    CGFloat boundsHeight = boundsSize.height;
    
    // 2.获取contentSize
    CGSize contentSize = self.scrollView.contentSize;
    CGFloat contentHeight = contentSize.height;
    // 3.保存原始偏移量，便于截图后复位
    CGPoint offset = self.scrollView.contentOffset;
    // 4.设置最初的偏移量为(0,0);
    [self.scrollView setContentOffset:CGPointMake(0, 0)];
    
    NSMutableArray *images = [NSMutableArray array];
    while (contentHeight > 0) {
        // 5.获取CGContext 5.获取CGContext
        UIGraphicsBeginImageContextWithOptions(boundsSize, NO, 0.0);
        CGContextRef ctx = UIGraphicsGetCurrentContext();
        // 6.渲染要截取的区域
        [self.layer renderInContext:ctx];
        UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        // 7.截取的图片保存起来
        [images addObject:image];
        
        CGFloat offsetY = self.scrollView.contentOffset.y;
        [self.scrollView setContentOffset:CGPointMake(0, offsetY + boundsHeight)];
        contentHeight -= boundsHeight;
    }
    // 8 webView 恢复到之前的显示区域
    [self.scrollView setContentOffset:offset];
    
    CGFloat scale = [UIScreen mainScreen].scale;
    
    CGSize imageSize = CGSizeMake(contentSize.width * scale,
                                  contentSize.height * scale);
    // 9.根据设备的分辨率重新绘制、拼接成完整清晰图片
    UIGraphicsBeginImageContext(imageSize);
    [images enumerateObjectsUsingBlock:^(UIImage *image, NSUInteger idx, BOOL *stop) {
        [image drawInRect:CGRectMake(0,
                                     scale * boundsHeight * idx,
                                     scale * boundsWidth,
                                     scale * boundsHeight)];
    }];
    
    UIImage *fullImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return fullImage;
}
```
想要体验这种方式的可以在`BLEDetailViewController`的`viewDidLoad`方法中,将导航栏右按钮的注释修改下：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    self.title = @"蓝牙详情";
    
//    UIBarButtonItem *rightItem = [[UIBarButtonItem alloc] initWithTitle:@"商品" style:UIBarButtonItemStylePlain target:self action:@selector(goToShopping)];
    
    UIBarButtonItem *rightItem = [[UIBarButtonItem alloc] initWithTitle:@"网络订单" style:UIBarButtonItemStylePlain target:self action:@selector(goToOrder)];
                                  
    self.navigationItem.rightBarButtonItem = rightItem;
    
    _infos = [[NSMutableArray alloc] init];
    _tableView.rowHeight = 60;
    
    //连接蓝牙并展示详情
    [self loadBLEInfo];
}
```

补充一些参数：
> 据佳博的一技术人员提供的一些参数：
> 汉字是24 x 24点阵，字符是12 x 24。
> 58mm 型打印机横向宽度384个点。(可是我用文字设置相对位置测试确实368，囧)
> 80mm 型打印机横向宽度576个点。
> 1mm 大概是8个点。


完整的库和Demo地址：[github地址](https://github.com/Halley-Wong/HLBluetoothDemo)

如果你只关注iOS 打印小票部分，不想太多操作蓝牙连接和处理，看这里：[蓝牙打印小票](https://github.com/Halley-Wong/SEBLEPrinter)

#  打印没反应？
首先，确定你使用的是标签打印机还是一般的小票打印机。

我写的Demo不支持标签打印机，你可以仿照我的例子，自己封装一下指令（我们并没有采购标签打印机，也没办法测试，抱歉了）。

如果你连接成功，但是发出打印指令后，打印机没反应，很有可能是因为你的打印机一次发送的数据长度小于146，你把146改的更小一点试试看。

我测试的两台佳博打印机，一台没有长度限制，一台最多每次只能发送146个字节，否则会出现打印没反应的情况，需要重启打印机。

不同的打印机，可能对长度的限制不太一样，据群友反应有的打印机只能支持一次发送20个字节，所以你需要将宏里面的146改的更小一点。

Have Fun !

