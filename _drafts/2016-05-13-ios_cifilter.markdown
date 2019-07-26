---
layout:     post
title:      "iOS 中的CIFilter(基础用法)"
date:       2016-05-13
author:     "Haley_Wong"
catalog:    true
tags:
    - CIFilter
---

本文大部分内容均来自：[Core Image Tutorial: Getting Started](https://www.raywenderlich.com/76285/beginning-core-image-swift)

Core Image 是一个很强大的库，PS图片时用到的各种滤镜就是在这个库中。而我们创建二维码、创建条形码用这里的滤镜，只需要短短几行代码就可以撸出来（后面会讲怎么用CIFilter绘制二维码、条形码）。

文中有提到在iOS 8 上，CIFilter 的API 里有126种滤镜可用，在 同时期 Mac OS 上有160多种滤镜可用；而在iOS 9.3 上，我测试可以使用的滤镜已经达到174种，Mac OS上肯定更多咯。

# 如何知道有哪些滤镜效果呢？
我一度想查找API里一共提供了多少种滤镜，每种滤镜分别有什么效果。可能是种类实在是太多，不同的滤镜又有很多不同的参数（参数名，参数各种都可能不同）设置，基本没有介绍每种滤镜的文章。

下面提供获取每种滤镜名称以及其属性的方法：
```
// swift 版
let properties = CIFilter.filterNamesInCategory(kCICategoryBuiltIn) 
println(properties)
for fileterName : String in properties {
    let filter = CIFilter(name: fileterName)
  // 滤镜的参数
    print(filter?.attributes)
}

// Objective-C版 （因转换成OC版的太简单，略😃）
```
# 准备工作
在iOS 中使用滤镜效果，需要用到的重要类有三个：
* **CIContext**. 图片的所有处理工作都是在 CIContext中做的. 它有点类似于 Core Graphics 和 OpenGL context.
* **CIImage**. 这个类持有图片数据。可以用UIImage或者图片路径或者data来创建一个CIImage对象。
* **CIFilter**.滤镜类，它有一个用来设置各种参数的字典，API已经提供了`setValue: forKey:`方法来设置参数。

# 基础用法
对一张图使用一个滤镜效果，总结起来需要四步：
1. **创建一个CIImage对象** .CImage 有很多初始化方法。譬如：CIImage(contentsOfURL:), CIImage(data:), CIImage(CGImage:), CIImage(bitmapData:bytesPerRow:size:format:colorSpace:),用的最多的是CIImage(contentsOfURL:)。
2. **创建一个CIContext**. CIContext 可能是基于CPU，也可能是基于GPU的。所以创建CIContext会消耗资源，影响性能，我们应该尽可能多的复用它。将处理过后的图片数据，输出为CIImage的时候会用到CIContext。
3. **创建一个滤镜**. 创建好滤镜后，我们需要为其设置参数。有的滤镜要设置的参数比较多，有的滤镜却不需要设置参数。
4. **获取filter output**. 滤镜会输出一个CIImage对象，用CIContext 可以将CIImage转换为UIImage。

这里有一段示例代码：
```
// 1.获取本地图片路径
let fileURL = NSBundle.mainBundle().URLForResource("image", withExtension: "png") 
// 2.创建CIImage对象
let beginImage = CIImage(contentsOfURL: fileURL!) 
// 3. 创建滤镜
// 创建一个棕榈色滤镜
let filter = CIFilter(name: "CISepiaTone")!
filter.setValue(beginImage, forKey: kCIInputImageKey)
// 设置输入的强度系数
filter.setValue(0.5, forKey: kCIInputIntensityKey)
// 4.将CIImage转换为UIImage 
//  其实在这个API内部用到了CIContext，而它就是在每次使用的使用去创建一个新的CIContext，比较影响性能
let newImage = UIImage(CIImage: filter.outputImage!)
self.imageView.image = newImage
```
> **注意点**
因为`let newImage = UIImage(CIImage: filter.outputImage)`内部会每次都创建一个CIContext，这里我们可以优化。用上面的方式创建的UIImage ，我们将其转换为NSData的时候，NSData为nil，原因是：`May return nil if image has no CGImageRef or invalid bitmap format`，这表明我们更应该优化。

将上面的第四步替换成如下代码：
```
        // 1 创建一个CIContext，只需要将这个context对象存起来，其他地方调用即可。
        let context = CIContext(options:nil)
        // 2 用CIContext将CIImage转换为CGImage
        let cgimg = context.createCGImage(filter.outputImage!, fromRect: filter.outputImage!.extent)
        // 3 将CGImage转换为UIImage
        let newImage = UIImage(CGImage: cgimg)
        self.imageView.image = newImage
```

# 将滤镜处理后的图片保存进相册
以前我们可能会用 `UIImageWriteToSavedPhotosAlbum()`。
ALAssetsLibrary 提供了将CGImage直接保存到相册的示例方法：`writeImageToSavedPhotosAlbum`,只可惜它到iOS 9.0 就弃用了☹️，当工程的最低兼容版本大于9.0时，编译器会给你一个警告，告诉你该用什么方法替换。

```
@IBAction func savePhoto(sender: UIButton) {
        // 1 获取滤镜输出的图片
        let imageToSave = filter.outputImage!
        // 2 创建一个使用CPU渲染器的CIContext
        let softwareContext = CIContext(options: [kCIContextUseSoftwareRenderer : true])
        // 3 将CIImage转换为CGImage
        let cgimage = softwareContext.createCGImage(imageToSave, fromRect: imageToSave.extent)
        // 4 使用 ALAssetsLibrary 保存到相册
        let library = ALAssetsLibrary()
        library.writeImageToSavedPhotosAlbum(cgimage, metadata: imageToSave.properties, completionBlock: nil) 
    }
```

# 组合滤镜
我们可以将多种滤镜效果组合起来，创建一个新的滤镜效果，这比将一个个的滤镜加到图片上，在输出要有效率的多。这里有一个示例：

```
    func oldPhoto(img: CIImage, withAmount intensity: Float) -> CIImage {
        
        // 1 创建一个棕色滤镜
        let sepiaFilter = CIFilter(name: "CISepiaTone")
        sepiaFilter?.setValue(img, forKey: kCIInputImageKey)
        sepiaFilter?.setValue(intensity, forKey: kCIInputIntensityKey)
        
        // 2 创建一个随机点滤镜
        let randomFilter = CIFilter(name: "CIRandomGenerator")
        
        // 3
        let lighten = CIFilter(name: "CIColorControls")
        lighten?.setValue(randomFilter?.outputImage, forKey: kCIInputImageKey)
        lighten?.setValue(1 - intensity, forKey: "inputBrightness")
        lighten?.setValue(0, forKey: "inputSaturation")
        
        // 4 将滤镜输出裁剪成原始图片大小
        let croppedImage = lighten?.outputImage?.imageByCroppingToRect(beginImage.extent)
        
        // 5
        let composite = CIFilter(name: "CIHardLightBlendMode")
        composite?.setValue(sepiaFilter?.outputImage, forKey: kCIInputImageKey)
        composite?.setValue(croppedImage, forKey: kCIInputBackgroundImageKey)
        
        // 6
        let vignette = CIFilter(name: "CIVignette")
        vignette?.setValue(composite?.outputImage, forKey: kCIInputImageKey)
        vignette?.setValue(intensity * 2, forKey: "inputIntensity")
        vignette?.setValue(intensity * 30, forKey: "inputRadius")
        
        // 7
        return (vignette?.outputImage)!
    }
```
![别人的图](/img/blogs/ios_cifilter/img_01.webp)

![代码效果图](/img/blogs/ios_cifilter/img_02.webp)

为毛自己家的效果图是这个鬼样子，别人家的效果图那么好看！😒 😒 


# 用滤镜效果创建二维码、条形码

**创建条形码**
 
```
+ (UIImage *)barCodeImageWithInfo:(NSString *)info
{
    // 创建条形码
    CIFilter *filter = [CIFilter filterWithName:@"CICode128BarcodeGenerator"];
    
    // 恢复滤镜的默认属性
    [filter setDefaults];
    // 将字符串转换成NSData
    NSData *data = [info dataUsingEncoding:NSUTF8StringEncoding];
    // 通过KVO设置滤镜inputMessage数据
    [filter setValue:data forKey:@"inputMessage"];
    // 获得滤镜输出的图像
    CIImage *outputImage = [filter outputImage];
    // 将CIImage 转换为UIImage
    UIImage *image = [UIImage imageWithCIImage:outputImage];
    
    // 如果需要将image转NSData保存，则得用下面的方式先转换为CGImage,否则NSData 会为nil
    //    CIContext *context = [CIContext contextWithOptions:nil];
    //    CGImageRef imageRef = [context createCGImage:outputImage fromRect:outputImage.extent];
    //
    //    UIImage *image = [UIImage imageWithCGImage:imageRef];
    
    return image;
}
```

![条形码](/img/blogs/ios_cifilter/img_03.webp)

**创建二维码**

```
+ (UIImage *)qrCodeImageWithInfo:(NSString *)info  width:(CGFloat)width
{
    if (!info) {
        return nil;
    }
    
    NSData *strData = [info dataUsingEncoding:NSUTF8StringEncoding allowLossyConversion:NO];
    //创建二维码滤镜
    CIFilter *qrFilter = [CIFilter filterWithName:@"CIQRCodeGenerator"];
    [qrFilter setValue:strData forKey:@"inputMessage"];
    [qrFilter setValue:@"H" forKey:@"inputCorrectionLevel"];
    CIImage *qrImage = qrFilter.outputImage;
    //颜色滤镜
    CIFilter *colorFilter = [CIFilter filterWithName:@"CIFalseColor"];
    [colorFilter setDefaults];
    [colorFilter setValue:qrImage forKey:kCIInputImageKey];
    [colorFilter setValue:[CIColor colorWithRed:0 green:0 blue:0] forKey:@"inputColor0"];
    
![Uploading 1A4978EE-427F-4804-B536-1D5C330A0578_306160.png . . .][colorFilter setValue:[CIColor colorWithRed:1 green:1 blue:1] forKey:@"inputColor1"];
    CIImage *colorImage = colorFilter.outputImage;
    //返回二维码
    CGFloat scale = width/31;
    UIImage *codeImage = [UIImage imageWithCIImage:[colorImage imageByApplyingTransform:CGAffineTransformMakeScale(scale, scale)]];
    return codeImage;
}
```

![二维码](/img/blogs/ios_cifilter/img_04.webp)

CIFilter初体验就先到这里了，Have Fun!

