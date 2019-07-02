---
layout:     post
title:      "iOS下50行代码实现图文混排"
date:       2015-08-24
author:     "Haley-Wong"
catalog:    true
tags:
    - Tools
---

## 开头
本文是技术集中的第一篇技术性文章，所以就记录一点简单且浅显易懂的东西。

现在即时通讯和朋友圈这两块功能基本上属于app的标配功能了吧。图文混排在这两块中使用最为常见，我已经做好了demo:[图文混排Demo](https://github.com/Haley-Wong/TextUtil)。

文中会讲述几点小技巧：图文混排、动态计算文字长度、图片拉伸方法。

## 以前的做法
在以前做图文混排的时候，经常使用`OHAttributedLabel`,后来苹果吸取了一些第三方的优点，对`NSString`做了扩展，作者也不再更新，推荐用系统的方法来实现图文混排。具体请自行百度或者google关键字`OHAttributedLabel`。

## 现在的做法
苹果在iOS7中推出了一个新的类`NSTextAttachment`，它是做图文混排的利器，本文就是用这个类，只用50行代码实现文字与表情混排，当然也可以实现段落中的图文混排，与CoreText比起来实在是简单了太多，下面讲述两个案例。(更新：YYKit中的YYTextAttachment，应该就是从NSTextAttachment上找的灵感吧)

### 案例一
先上效果图，聊天界面中的图文混排
    
![](/img/blogs/photoText/img_01.jpg)

要实现这样的效果，code4app上似乎有很多种做法，还有一些奇葩的一个字符一个label，但是今天要讲述的做法，是目前为止我看到的最简单的做法了，只用一个UILabel,需要用到UILabel的attributedText属性。

首先，需要组装一个表情和文字对应的plist文件，plist中的键值对如下：

![](/img/blogs/photoText/img_02.jpg)

本文用一个工具类来实现一个转换的方法，你也可以给NSString添加一个类别来实现。

第一步，解析plist文件，转化为数组。

```objc
NSString *filePath = [[NSBundle mainBundle] pathForResource:@"emoticons" ofType:@"plist"];
NSArray *face = [NSArray arrayWithContentsOfFile:filePath];
```
第二步，将字符串转换为可变属性字符串，并通过正则表达式匹配出所有的要替换的字符。

```objc
// 1、创建一个可变的属性字符串

NSMutableAttributedString *attributeString = [[NSMutableAttributedString alloc] initWithString:text];
// 2、通过正则表达式来匹配字符串

NSString *regex_emoji = @"\\[[a-zA-Z0-9\\/\\u4e00-\\u9fa5]+\\]";//匹配表情
NSError *error =nil;
NSRegularExpression *re = [NSRegularExpression regularExpressionWithPattern:regex_emoji options:NSRegularExpressionCaseInsensitive error:&error];
if (!re) {
    NSLog(@"%@", [errorlocalizedDescription]);
    return attributeString;
}
NSArray *resultArray = [rematchesInString:text options:0 range:NSMakeRange(0, text.length)];
```
数组中都是NSTextCheckingResult对象，它包含了特殊字符在整个字符串中的位置等信息。

第三步，将特殊字符与对应表情关联
```
NSMutableArray *imageArray = [NSMutableArray arrayWithCapacity:resultArray.count];
//根据匹配范围来用图片进行相应的替换
for(NSTextCheckingResult *match in resultArray) {
    //获取数组元素中得到range
    NSRangerange = [match range];
    //获取原字符串中对应的值
    NSString*subStr = [text substringWithRange:range];
    for(inti =0; i < face.count; i ++) {
        if ([face[i][@"cht"] isEqualToString:subStr]) {
            //face[i][@"png"]就是我们要加载的图片
            //新建文字附件来存放我们的图片,iOS7才新加的对象
            NSTextAttachment*textAttachment = [[NSTextAttachment alloc] init];
            //给附件添加图片
            textAttachment.image = [UIImage imageNamed:face[i][@"png"]];
            //调整一下图片的位置,如果你的图片偏上或者偏下，调整一下bounds的y值即可
            textAttachment.bounds = CGRectMake(0, -8, textAttachment.image.size.width, textAttachment.image.size.height);
            //把附件转换成可变字符串，用于替换掉源字符串中的表情文字
            NSAttributedString*imageStr = [NSAttributedString attributedStringWithAttachment:textAttachment];
            //把图片和图片对应的位置存入字典中
            NSMutableDictionary*imageDic = [NSMutableDictionary dictionaryWithCapacity:2];
            [imageDic setObject:imageStr forKey:@"image"];
            [imageDic setObject:[NSValuevalueWithRange:range] forKey:@"range"];
            //把字典存入数组中
            [imageArray addObject:imageDic];
        }
    }
}
```
第四步，将特殊字符替换成图片
```objc
// 4、从后往前替换，否则会引起位置问题
for (int i = (int)imageArray.count-1; i >=0; i--) {
    NSRange range;
    [imageArray[i][@"range"] getValue:&range];
    //进行替换
    [attributeString replaceCharactersInRange:range withAttributedString:imageArray[i][@"image"]];
}
```

用法：
```objc
NSString *content = @"文字加上表情[得意][酷][呲牙]";
NSMutableAttributedString *attrStr = [Utility emotionStrWithString:content];
_contentLabel.attributedText= attrStr;
```

### 案例二

![](/img/blogs/photoText/img_03.jpg)

需要实现的效果：

有了上面的方法，这个效果更容易实现，只需要将某些图片给它设置一个固定的字符对应即可。

与以上方法主要不同点在于正则表达式：
```objc
//2、匹配字符串
NSError *error = nil;
NSRegularExpression *re = [NSRegularExpression regularExpressionWithPattern:string options:NSRegularExpressionCaseInsensitive error:&error];
if (!re) {
    NSLog(@"%@", [error localizedDescription]);
    return attributeString;
}
```

用法：
```objc
NSString *praiseStr = @"路人甲、路人乙";
NSString *praiseInfo = [NSStringstringWithFormat:@"<点赞> %@",praiseStr];
NSDictionary *attributesForAll = @{NSFontAttributeName:[UIFontsystemFontOfSize:14.0],NSForegroundColorAttributeName:[UIColorgrayColor]};

NSMutableAttributedString *attrStr = [Utility exchangeString:@"<点赞>" withText:praiseInfoimageName:@"dynamic_love_blue"];
```

## 彩蛋

1、计算动态文字的长度
```objc
NSMutableAttributedString *content = [Utility emotionStrWithString:_dynamic.text];
[content addAttribute:NSFontAttributeName value:kContentFont  range:NSMakeRange(0, content.length)];
CGSize maxSize = CGSizeMake(kDynamicWidth,MAXFLOAT);
CGSize attrStrSize = [content boundingRectWithSize:maxSize options:NSStringDrawingUsesLineFragmentOrigin context:nil].size;
```

其中NSMutableAttributedString类型的字符串可以添加多种属性，并且在计算的时候必须设置字符大小等属性。

2、图片拉伸


在iOS5之前可以用`stretchableImageWithLeftCapWidth: topCapHeight:`

iOS5之中用`resizableImageWithCapInsets:`

iOS6开始多了一个参数`resizableImageWithCapInsets:resizingMode:`

