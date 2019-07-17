---
layout:     post
title:      "SDWebImageV3.7.5源码解析"
date:       2016-04-15
author:     "Haley_Wong"
catalog:    true
tags:
    - SDWebImage
---

SDWebImage更新到如今这个版本，过程做了许多改进，性能已经非常的好了。以前就粗略的看过SDWebImage的源码，但是未做记录整理。每次阅读还是受益良多，故做此记录。SDWebImage的结构比较混乱，所以解析其调用顺序也是相当的绕，比较难以理解。

## SDWebImage使用场景
SDWebImage通过添加category的方式，为UIImageView、UIButton、MKAnnotationView 扩展设置网络图片的方法。使用方式基本类似，本文就拿UIImageView来举例：
![123.png](http://upload-images.jianshu.io/upload_images/727768-8007e5acdf102d60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而上面几个方法在实现时，都是调用的最后这个方法，只是设置了一些默认参数。

例如：

```
- (void)sd_setImageWithURL:(NSURL *)url {
    [self sd_setImageWithURL:url placeholderImage:nil options:0 progress:nil completed:nil];
}
```

## 源码解析

```
- (void)sd_setImageWithURL:(NSURL *)url
          placeholderImage:(UIImage *)placeholder
                   options:(SDWebImageOptions)options
                  progress:(SDWebImageDownloaderProgressBlock)progressBlock
                 completed:(SDWebImageCompletionBlock)completedBlock;
```

下面粗略概括上面这个方法的实现过程，加括号说明的是需要抽出来详细解析的：
> 
* 1.取消该UIImageView的当前图片加载操作。（内部实现值得详细解析)
* 2.利用runtime的关联对象AssociatedObject为该UIImageView设置网络图片的url。（runtime的使用场景)
* 3.设置默认图片。（即placeholder，若设置了延迟设置placeholder，则跳过该步）
* 4.判断url是否存在，不存在则回调completedBlock，返回错误信息；若存在，执行下一步。
* 5.判断是否添加ActivityIndicatorView。
* 6.调用SDWebImageManager，创建下载图片的operation。（这一步是重点）
* 7.为该UIImageView设置下载的operation。（同样是通过runtime的关联对象AssociatedObject实现）
* 8.执行下载完成的completedBlock回调。（这一步也值得详细解析）

### 重点一

取消UIImageView的当前图片加载操作。为什么需要取消当前加载操作呢？

举个例子，我为imageView设置了网络图片1，然后它去下载网络图片了，因为下载可能需要一段时间，而且下载过程是异步的。如果还没下载完，我又为其设置了网络图片2，这时候会出现多种问题：
* 1.网络图片2先下载完，显示为网络图片2，而网络图片1下载完，又显示成网络图片1。
* 2.网络图片1先下载完，显示为图片1后，网络图片2下载完后，又变换为图片2。
* 3.而设置图片2之后，下载图片1的流量以及设置的资源损耗都是不必要的。

所以如果当前有图片正在下载的话，先取消掉当前的图片加载。

```
- (void)sd_cancelCurrentImageLoad {
    [self sd_cancelImageLoadOperationWithKey:@"UIImageViewImageLoad"];
}

- (void)sd_cancelCurrentAnimationImagesLoad {
    [self sd_cancelImageLoadOperationWithKey:@"UIImageViewAnimationImages"];
}
```

可以看到，这里有两个不同的取消方法，因为UIImageView除了可以设置单张图片，还可以设置多张网络图片展示动画效果。这两个方法内部调用的是同一个方法：

```
- (void)sd_cancelImageLoadOperationWithKey:(NSString *)key {
    // Cancel in progress downloader from queue
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    id operations = [operationDictionary objectForKey:key];
    if (operations) {
        if ([operations isKindOfClass:[NSArray class]]) {
            for (id <SDWebImageOperation> operation in operations) {
                if (operation) {
                    [operation cancel];
                }
            }
        } else if ([operations conformsToProtocol:@protocol(SDWebImageOperation)]){
            [(id<SDWebImageOperation>) operations cancel];
        }
        [operationDictionary removeObjectForKey:key];
    }
}
```
上面这个方法是UIView的一个category方法，在`UIView + WebCacheOperation`中。

第一行`[self operationDictionary]`也是用runtime的AssociatedObject获取当前视图的operationDictionary，没有的话就创建一个set上去，具体实现如下：
```
- (NSMutableDictionary *)operationDictionary {
    NSMutableDictionary *operations = objc_getAssociatedObject(self, &loadOperationKey);
    if (operations) {
        return operations;
    }
    operations = [NSMutableDictionary dictionary];
    objc_setAssociatedObject(self, &loadOperationKey, operations, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    return operations;
}
```

后面几行内容就是取消掉当前operation的下载操作。

因为可能是UIImageView的动画图片，所以就去数组中一个个的取消。

如果是SDWebImage自定义的对象肯定会实现自定义的取消协议，则转换对象后取消。

否则直接将这个object从字典中删除。

至此，取消当前图片下载步骤完毕。

### 重点二

调用SDWebImageManager，创建下载图片的operation。
```
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;
```
该方法的内部实现是SDWebImage的核心，所有的精华都在这里。

实现中多次使用`dispatch_main_sync_safe` 和`dispatch_main_async_safe`。他们俩分别对应两个宏，一是为防止在主线程执行主线程操作发生死锁；二是避免不必要的开销。`dispatch_async`不管怎么说都会有一定的开销吧。

```
#define dispatch_main_sync_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_sync(dispatch_get_main_queue(), block);\
    }

#define dispatch_main_async_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }
```
#### 第一步
验证url，如果是字符串转换为NSURL，如果不是NSURL类型，url置为nil。
```
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // Prevents app crashing on argument type error like sending NSNull instead of NSURL
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }
```
#### 第二步

创建一个SDWebImageCombinedOperation对象，代表一个图片加载任务，但是实际下载图片的事是由另一个Operation来做，该类也实现了SDWebImageOperation协议。因为可能会在block中调用operation，所以先处理处理好循环引用问题。

```
__block SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
__weak SDWebImageCombinedOperation *weakOperation = operation;
```
#### 第三步
判断url是否正确，如果url有问题，则直接返回包含错误信息的completeBlock。

首先，判断failedURLs中是否包含该url，如果包含则是错误的url。该步骤可能会出现多线程读取问题，所以添加`@synchronized`同步锁。

然后，判断url的绝对路径是否存在，结合上面结果分析是否错误。

```
BOOL isFailedUrl = NO;
    @synchronized (self.failedURLs) {
        isFailedUrl = [self.failedURLs containsObject:url];
    }

if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        dispatch_main_sync_safe(^{
            NSError *error = [NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil];
            completedBlock(nil, error, SDImageCacheTypeNone, YES, url);
        });
        return operation;
    }
```
#### 第四步

将operation加进数组中，需要添加同步锁，保证数组的读写安全。

```
@synchronized (self.runningOperations) {
        [self.runningOperations addObject:operation];
    }
```
#### 第五步
获取该网络图片缓存用的key。

`NSString *key = [self cacheKeyForURL:url];` 展开这个方法是：

```
- (NSString *)cacheKeyForURL:(NSURL *)url {
    if (self.cacheKeyFilter) {
        return self.cacheKeyFilter(url);
    }
    else {
        return [url absoluteString];
    }
}
```
这里如果给manager设置过cacheKeyFilter，则会按照自己的设置返回一个字符串作为key，否则会直接返回url 的绝对路径`absoluteString `。

#### 第六步
```
operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:^(UIImage *image, SDImageCacheType cacheType) {
```
这一步是为上面第二步创建的operation对象设置cacheOperation。
```
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock {
    if (!doneBlock) {
        return nil;
    }

    if (!key) {
        doneBlock(nil, SDImageCacheTypeNone);
        return nil;
    }

    // First check the in-memory cache...
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        doneBlock(image, SDImageCacheTypeMemory);
        return nil;
    }

    NSOperation *operation = [NSOperation new];
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) {
            return;
        }

        @autoreleasepool {
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage && self.shouldCacheImagesInMemory) {
                NSUInteger cost = SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });

    return operation;
}
```
> 在下载图片前先获取缓存的操作就是这里。

该方法有两个参数，第一个参数传key，第二个参数是个block，是从本地取出缓存的图片后的回调。内部实现部分分析在下面

##### 6.1 

判断参数是否完整，否则直接返回cacheOperation为nil。

##### 6.2 

先从内存中查找缓存的图片，若找到，则调用doneBlock，返回图片和缓存图片方式，该方法返回nil。
```
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key {
  // 此处的memCache是NSCache。
    return [self.memCache objectForKey:key];
}
```

##### 6.3 

创建一个operation，并开启一个子线程，从本地磁盘查找图片，方法直接返回operation。该步骤内部实现就比较复杂了，需要详细的解析了。

**解析从磁盘获取缓存图片:**

```
- (UIImage *)diskImageForKey:(NSString *)key {
    NSData *data = [self diskImageDataBySearchingAllPathsForKey:key];
    if (data) {
        UIImage *image = [UIImage sd_imageWithData:data];
        image = [self scaledImageForKey:key image:image];
        if (self.shouldDecompressImages) {
            image = [UIImage decodedImageWithImage:image];
        }
        return image;
    }
    else {
        return nil;
    }
}
```

第一行，从所有磁盘缓存路径中查找key所对应的图片。首先从默认的缓存路径下查找，默认缓存路径是项目的Caches/default/com.hackemist.SDWebImageCache.default。

其中default/com.hackemist.SDWebImageCache.default是SDWebImage创建的。如果没找到，再从其他我们自定义的缓存路径下查找。

> 这里的key(即网络图片的完整路径)，需要将其进行MD5加密，然后图片在本地的名称就是加密后的名称。
> (如加密前的key是`http://www.haley.cn/file-service/image/user/photo/eb68e7030000857f.jpg`，加密后是`8269e32aad025d366379b1d11f279729`)

第三行，将从磁盘路径上获取的NSData，转换为UIImage。

```
+(UIImage *)sd_imageWithData:(NSData *)data {
    if (!data) {
        return nil;
    }
    UIImage *image;
  // 根据data 判断image的类型：jpeg、png、gif、tiff、webp
    NSString *imageContentType = [NSData sd_contentTypeForImageData:data];
    if ([imageContentType isEqualToString:@"image/gif"]) {
        image = [UIImage sd_animatedGIFWithData:data];
    }
#ifdef SD_WEBP
    else if ([imageContentType isEqualToString:@"image/webp"])
    {
        image = [UIImage sd_imageWithWebPData:data];
    }
#endif
    else {
        image = [[UIImage alloc] initWithData:data];
        UIImageOrientation orientation = [self sd_imageOrientationFromImageData:data];
        if (orientation != UIImageOrientationUp) {
            image = [UIImage imageWithCGImage:image.CGImage
                                        scale:image.scale
                                  orientation:orientation];
        }
    }

    return image;
}
```

这里gif和webp格式的图片需要特殊处理，其他格式的图片需要判断下方向，然后设置正确图片方向。
第四行，将图片根据设备的屏幕品质，进行缩放处理，返回发缩放后的图片。

第五六行，如果shouldDecompressImages为YES，默认就是为YES，表示是否解码图片，NSData转换的image，会在第一次渲染到屏幕上的时候才进行解码，并且每次从NSData读取时，都需要解码一次，这个过程苹果没做过优化，所以可能会造成卡顿。

关于图片的缓存和解码可以看这里：[iOS 处理图片的一些小 Tip](http://blog.ibireme.com/2015/11/02/ios_image_tips/)

关于图片的解码过程可以看这篇C语言文章：[JPEG图像的解压缩操作](http://www.cnblogs.com/hzhida/archive/2012/05/30/2524989.html)

##### 6.4 
将解码后的图片保存到缓存memCache中，便于以后直接从缓存中获取。
##### 6.5
回调doneBlock，返回图片和缓存类型。

#### 第七步
在cacheOperation的doneBlock中。如果图片取到了缓存图片，则直接将图片等信息通过completedBlock返回。

从runningOperation中删除步骤二中创建的该operation。

```
dispatch_main_sync_safe(^{
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (strongOperation && !strongOperation.isCancelled) {
                    completedBlock(image, nil, cacheType, YES, url);
                }
            });
            @synchronized (self.runningOperations) {
                [self.runningOperations removeObject:operation];
            }
```
如果返回的图片为nil，并且需要下载，则创建下载图片的operation
```
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageDownloaderOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageDownloaderCompletedBlock)completedBlock;
```
下载过程待会详细分析，这里先分析下载完成后的操作。

情形一：回调返回的error，如果不为空，则返回错误给completedBlock。如果url有问题，则把url添加到failedURLs中。

情形二：如果成功，则先从failedURLs中删除url，里面不包含也没关系。
如果url对应的图片是url不变，但是图片会变的，则不缓存。

如果图片需要转换，则将图片转换后保存到内存和磁盘中，调用block返回图片。

```
UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];
```
```
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk {
    if (!image || !key) {
        return;
    }
    // if memory cache is enabled
    if (self.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(image);
        [self.memCache setObject:image forKey:key cost:cost];
    }

    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            NSData *data = imageData;

            if (image && (recalculate || !data)) {
#if TARGET_OS_IPHONE
                // We need to determine if the image is a PNG or a JPEG
                // PNGs are easier to detect because they have a unique signature (http://www.w3.org/TR/PNG-Structure.html)
                // The first eight bytes of a PNG file always contain the following (decimal) values:
                // 137 80 78 71 13 10 26 10

                // If the imageData is nil (i.e. if trying to save a UIImage directly or the image was transformed on download)
                // and the image has an alpha channel, we will consider it PNG to avoid losing the transparency
                int alphaInfo = CGImageGetAlphaInfo(image.CGImage);
                BOOL hasAlpha = !(alphaInfo == kCGImageAlphaNone ||
                                  alphaInfo == kCGImageAlphaNoneSkipFirst ||
                                  alphaInfo == kCGImageAlphaNoneSkipLast);
                BOOL imageIsPng = hasAlpha;

                // But if we have an image data, we will look at the preffix
                if ([imageData length] >= [kPNGSignatureData length]) {
                    imageIsPng = ImageDataHasPNGPreffix(imageData);
                }

                if (imageIsPng) {
                    data = UIImagePNGRepresentation(image);
                }
                else {
                    data = UIImageJPEGRepresentation(image, (CGFloat)1.0);
                }
#else
                data = [NSBitmapImageRep representationOfImageRepsInArray:image.representations usingType: NSJPEGFileType properties:nil];
#endif
            }

            if (data) {
                if (![_fileManager fileExistsAtPath:_diskCachePath]) {
                    [_fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
                }

                // get cache Path for image key
                NSString *cachePathForKey = [self defaultCachePathForKey:key];
                // transform to NSUrl
                NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];

                [_fileManager createFileAtPath:cachePathForKey contents:data attributes:nil];

                // disable iCloud backup
                if (self.shouldDisableiCloud) {
                    [fileURL setResourceValue:[NSNumber numberWithBool:YES] forKey:NSURLIsExcludedFromBackupKey error:nil];
                }
            }
        });
    }
}
```
如果不需要转换，则直接保存和回调block。
```
if (downloadedImage && finished) {
                            [self.imageCache storeImage:downloadedImage recalculateFromImage:NO imageData:data forKey:key toDisk:cacheOnDisk];
                        }

                        dispatch_main_sync_safe(^{
                            if (strongOperation && !strongOperation.isCancelled) {
                                completedBlock(downloadedImage, nil, SDImageCacheTypeNone, finished, url);
                            }
                        });
```

##### 下载过程解析
在downloader中有一个`URLCallbacks`的可变字典，每一个url作为key，对应一个数组（数组中是字典对象，字典中保存下载operation的progressBlock和completeBlock），然后判断该url是否是首次下载，如果是，则调用创建operation的block，否则直接返回没有初始化的operation（nil）。

```
- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return;
    }

    dispatch_barrier_sync(self.barrierQueue, ^{
        BOOL first = NO;
        if (!self.URLCallbacks[url]) {
            self.URLCallbacks[url] = [NSMutableArray new];
            first = YES;
        }

        // Handle single download of simultaneous download request for the same URL
        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;

        if (first) {
            createCallback();
        }
    });
}
```
createCallBack内部创建operation的过程是先创建一个NSMutableURLRequest，需要保证该url不被缓存。

过程如下：
```
NSTimeInterval timeoutInterval = wself.downloadTimeout;
        if (timeoutInterval == 0.0) {
            timeoutInterval = 15.0;
        }
        NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];
        request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
        request.HTTPShouldUsePipelining = YES;
        if (wself.headersFilter) {
            request.allHTTPHeaderFields = wself.headersFilter(url, [wself.HTTPHeaders copy]);
        }
        else {
            request.allHTTPHeaderFields = wself.HTTPHeaders;
        }
```
然后创建一个operation对象：
```
- (id)initWithRequest:(NSURLRequest *)request
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock;
```

然后将下载operation存进downloadQueue中，并判断是否设置过后进先出的执行顺序，默认是先进先出的执行。

```
if (options & SDWebImageDownloaderHighPriority) {
            operation.queuePriority = NSOperationQueuePriorityHigh;
        } else if (options & SDWebImageDownloaderLowPriority) {
            operation.queuePriority = NSOperationQueuePriorityLow;
        }

        [wself.downloadQueue addOperation:operation];
        if (wself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
            // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
            [wself.lastAddedOperation addDependency:operation];
            wself.lastAddedOperation = operation;
        }
```

##### SDWebImageDownloaderOperation
自定义初始化方法
```
- (id)initWithRequest:(NSURLRequest *)request
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock {
    if ((self = [super init])) {
        _request = request;
        _shouldDecompressImages = YES;
        _shouldUseCredentialStorage = YES;
        _options = options;
        _progressBlock = [progressBlock copy];
        _completedBlock = [completedBlock copy];
        _cancelBlock = [cancelBlock copy];
        _executing = NO;
        _finished = NO;
        _expectedSize = 0;
        responseFromCached = YES; // Initially wrong until `connection:willCacheResponse:` is called or not called
    }
    return self;
}
```
~~然后重写start方法，start方法内部创建NSURLConnection，通过NSURLConnection来下载图片，利用connection的代理方法来更新progressBlock的进度和完成的block。~~

当然，随着版本更迭，SDWebImage V3.8.0之后（包括V3.8.0）已经将NSURLConnection换成了NSURLSession，然后利用NSURLSession 的代理方法来更新progressBlock 和 completionHandler。还会在不同的结果时，发送通知。

> 下载完成后，也需要将NSData转换成的UIImage进行解码处理。

