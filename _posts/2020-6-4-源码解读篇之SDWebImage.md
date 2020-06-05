---
layout:     post
title:      SDWebImage源码解读
subtitle:   SDWebImage源码解读与分析
date:       2020-6-4
author:     eyuxin
header-img: 
catalog: true
tags:
    - iOS
    - SDWebImage
---



[SDWebImage](https://github.com/SDWebImage/SDWebImage)是我们常用的图片缓存加载库，基本是iOS项目中标配的第三方库。SDWebImage提供了图片从加载、解析、处理、缓存、清理等一系列功能。这篇文章只是一个概括。

## 整体流程

1.  sd_setImageWithURL()  

    入口函数, 是UIView的一个分类, 会调用setImageWithURL:placeholderImage:options:方法.

    先显示占位图, 然后交给SDWebImageManager根据URL开始处理图片

2.  SDImageCache类先从内存缓存查找是否有图片缓存，如果内存中已经有图片缓存，则直接回调到前端进行图片的显示。
3.  如果内存缓存中没有，则生成NSInvocationOperation添加到队列开始从硬盘中查找图片是否已经缓存。根据url为key在硬盘缓存目录下尝试读取图片文件，这一步是在NSOperation下进行的操作，所以需要回到主线程进行查找结果的回调。如果从硬盘读取到了图片，则将图片添加到内存缓存中，然后再回调到前端进行图片的显示。如果从硬盘缓存目录读取不到图片，说明所有缓存都不存在该图片，则需要下载图片。
4.  共享或重新生成一个下载器SDWebImageDownloader开始下载图片。图片的下载由NSURLConnection来处理，实现相关delegate来判断的下载状态：下载中、下载完成和下载失败。
5.  图片数据下载完成之后，交给SDWebImageDecoder类做图片解码处理，图片的解码处理在NSOperationQueue完成，不会阻塞主线程。在图片解码完成后，会回调给SDWebImageDownloader，然后回调给SDWebImageManager告知图片下载完成，通知所有的downloadDelegates下载完成，回调给需要的地方显示图片。
6.  最后将图片通过SDImageCache类，同时保存到内存缓存和硬盘缓存中。写文件到硬盘的过程也在以单独NSInvocationOperation完成，避免阻塞主线程。



## 缓存部分

##### 内存缓存部分

1.  实现原理

    使用SDMemoryCache这个类, 内部就是将 **NSCache** 扩展为了 SDMemoryCache 协议，并加入了 **NSMapTable \*weakCache** ，并为其添加了信号量锁来保证线程安全。weakCache 使用的是 strong-weak 引用不会有有额外的内存开销且不影响对象的生命周期。

    weakCache 的作用在于恢复缓存，它通过 CacheConfig 的 **shouldUseWeakMemoryCache** 开关以控制, 默认是打开的.

    

    由于 NSCache 遵循 `NSDiscardableContent` 策略来存储临时对象的，当内存紧张时，缓存对象有可能被系统清理掉。此时，如果应用访问 MemoryCache 时，缓存一旦未命中，则会转入 diskCache 的查询操作，可能导致 image 闪烁现象。而使用 weakCache 后, 由于weakCache保存着对象的弱引用 （在对象 被 NSCache 被清理且没有被释放的情况下)，我们可通过 weakCache 取到缓存，将其塞回 NSCache 中。从而减少磁盘 I/O。

2.  清理机制

    NSCache本身就支持清理机制, 在初始化之前设置它的totalCostLimit以及countLimit, 超过这个值之后就会清理了, 另外，在 NSCache 子类的初始化方法中监听了系统内存警告的通知，当系统收到内存警告时，清空内存中所有的对象。

##### 磁盘缓存部分:

1.  实现原理

    本质上就是在用户指定缓存目录后, 对图片进行I/O操作, 其中用到了一个ioQueue的串行队列, 采用异步的方式进行I/O操作.

2.  清理机制 (清理时机+LRU机制)

    首先监听了程序杀死以及进入后台的两个通知

    ```objective-c
    [[NSNotificationCenter defaultCenter] addObserver:self
                                    selector:@selector(deleteOldFiles)
                                    name:UIApplicationWillTerminateNotification
                                    object:nil];
            [[NSNotificationCenter defaultCenter] addObserver:self
                            selector:@selector(backgroundDeleteOldFiles)
                            name:UIApplicationDidEnterBackgroundNotification
                            object:nil];
    ```

    >   Tip: 在进入后台时需要使用[UIApplication.sharedApplication beginBackgroundTaskWithExpirationHandler] 方法来执行代码.

    

    过期的缓存文件全部删除，缓存总大小超过 maxSize时，从最老的文件开始删除直到当前缓存大小在 maxSize/2 值以下。

    这里的**LRU**机制实现原理是:  用NSURLContentModificationDateKey将本地的url排序, 然后优先删除时间比较久的文件



## 下载部分

#### SDWebImageDownloader

SDWebImageDownloader是一个单例对象，主要功能可以概括为：

-   通过SDWebImageDownloaderOptions枚举来设置图片从网络加载的不同情况
-   定义并管理了NSURLSession对象，通过这个对象来做网络请求，并且实现对象的代理方法
-   定义了一个NSURLRequest对象，并且管理请求头的拼装
-   对于每一个网络请求，通过一个SDWebImageDownloaderOperation自定义的NSOperation来操作网络下载
-   管理网络加载过程和完成时的回调工作。

同时设置了缓存策略, 过期时间, 是否解压缩返回的图片, 指定优先级, SSL验证 ,下载进度等.

#### SDWebImageDownloaderOperation

SDWebImageDownloaderOperation继承自NSOperation，负责生成NSURLSessionTask进行图片请求，支持下载取消和后台下载，在下载时及时反馈下载进度，在下载成功后，对图片进行解码，缩放和压缩等操作。



##  图片解码部分

#### SDWebImageCodersManager

编码解码管理器，处理多个图片编码解码任务，编码器是一个优先队列，这意味着后面添加的编码器将具有最高优先级。

相关方法

```objective-c
/// 添加编码器
- (void)addCoder:(nonnull id<SDWebImageCoder>)coder {
    /// 判断该编码器是否实现了SDWebImageCode协议
    if ([coder conformsToProtocol:@protocol(SDWebImageCoder)]) {
        dispatch_barrier_sync(self.mutableCodersAccessQueue, ^{
            [self.mutableCoders addObject:coder];
        });
    }
}

/// 删除编码器
- (void)removeCoder:(nonnull id<SDWebImageCoder>)coder {
    /// 同步删除编码器
    dispatch_barrier_sync(self.mutableCodersAccessQueue, ^{
        [self.mutableCoders removeObject:coder];
    });
}

/// 判断该图片是否可用编码
- (BOOL)canEncodeToFormat:(SDImageFormat)format {
    for (id<SDWebImageCoder> coder in self.coders) {
        if ([coder canEncodeToFormat:format]) {
            return YES;
        }
    }
    return NO;
}

/// 图像解码方法
- (UIImage *)decodedImageWithData:(NSData *)data {
    if (!data) {
        return nil;
    }
    for (id<SDWebImageCoder> coder in self.coders) {
        if ([coder canDecodeFromData:data]) {
            return [coder decodedImageWithData:data];
        }
    }
    return nil;
}

/// 压缩图像
- (UIImage *)decompressedImageWithImage:(UIImage *)image
                                   data:(NSData *__autoreleasing  _Nullable *)data
                                options:(nullable NSDictionary<NSString*, NSObject*>*)optionsDict {
    if (!image) {
        return nil;
    }
    for (id<SDWebImageCoder> coder in self.coders) {
        if ([coder canDecodeFromData:*data]) {
            return [coder decompressedImageWithImage:image data:data options:optionsDict];
        }
    }
    return nil;
}

/// 根据image和format（类型）编码图像
- (NSData *)encodedDataWithImage:(UIImage *)image format:(SDImageFormat)format {
    if (!image) {
        return nil;
    }
    for (id<SDWebImageCoder> coder in self.coders) {
        if ([coder canEncodeToFormat:format]) {
            return [coder encodedDataWithImage:image format:format];
        }
    }
    return nil;
}
```



#### SDWebImageImageIOCoder 主要的coder

通过该类可以实现图片的解压缩操作，对于太大的图片，先按照一定比例缩小图片，然后再进行解压缩。

```objective-c
#if SD_UIKIT || SD_WATCH
// 每个像素占用的字节数
static const size_t kBytesPerPixel = 4;
// 色彩空间占用的字节数
static const size_t kBitsPerComponent = 8;
// 定义一张图片可以占用的最大空间
static const CGFloat kDestImageSizeMB = 60.0f;

static const CGFloat kSourceImageTileSizeMB = 20.0f;

static const CGFloat kBytesPerMB = 1024.0f * 1024.0f;
// 1MB可以存储多少像素
static const CGFloat kPixelsPerMB = kBytesPerMB / kBytesPerPixel;
// 如果像素小于这个值，则不解压缩
static const CGFloat kDestTotalPixels = kDestImageSizeMB * kPixelsPerMB;
static const CGFloat kTileTotalPixels = kSourceImageTileSizeMB * kPixelsPerMB;

static const CGFloat kDestSeemOverlap = 2.0f;
#endif

/// 判断是否可以解码
- (BOOL)canDecodeFromData:(nullable NSData *)data {
    switch ([NSData sd_imageFormatForImageData:data]) {
        case SDImageFormatWebP:
            // WebP类型不支持解码
            return NO;
        case SDImageFormatHEIC:
            // 检查HEIC解码的兼容性
            return [[self class] canDecodeFromHEICFormat];
        default:
            return YES;
    }
}

/// 判断是否可以解码，并且实现逐渐显示效果
- (BOOL)canIncrementallyDecodeFromData:(NSData *)data {
    switch ([NSData sd_imageFormatForImageData:data]) {
        case SDImageFormatWebP:
            // 不支持WebP渐进解码
            return NO;
        case SDImageFormatHEIC:
            // Check HEIC decoding compatibility
            return [[self class] canDecodeFromHEICFormat];
        default:
            return YES;
    }
}

/// 解压缩图片
- (nullable UIImage *)sd_decompressedImageWithImage:(nullable UIImage *)image {
    // 判断图片是否能解压缩
    if (![[self class] shouldDecodeImage:image]) {
        return image;
    }
    
    // autorelease the bitmap context and all vars to help system to free memory when there are memory warning.
    // on iOS7, do not forget to call [[SDImageCache sharedImageCache] clearMemory];
    // 解压缩操作放入一个自动释放池里面，以便自动释放所有的变量
    @autoreleasepool{
        
        CGImageRef imageRef = image.CGImage;
        // 获取图片的色彩空间
        CGColorSpaceRef colorspaceRef = [[self class] colorSpaceForImageRef:imageRef];
        
        size_t width = CGImageGetWidth(imageRef);
        size_t height = CGImageGetHeight(imageRef);
        
        // kCGImageAlphaNone is not supported in CGBitmapContextCreate.
        // Since the original image here has no alpha info, use kCGImageAlphaNoneSkipLast
        // to create bitmap graphics contexts without alpha info.
        CGContextRef context = CGBitmapContextCreate(NULL,
                                                     width,
                                                     height,
                                                     kBitsPerComponent,
                                                     0,
                                                     colorspaceRef,
                                                     kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
        if (context == NULL) {
            return image;
        }
        
        // Draw the image into the context and retrieve the new bitmap image without alpha
        // 绘制一个和图片大小一样的图片
        CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
        // 创建一个没有alpha通道的图片
        CGImageRef imageRefWithoutAlpha = CGBitmapContextCreateImage(context);
        // 得到解压缩以后的图片
        UIImage *imageWithoutAlpha = [[UIImage alloc] initWithCGImage:imageRefWithoutAlpha scale:image.scale orientation:image.imageOrientation];
        CGContextRelease(context);
        CGImageRelease(imageRefWithoutAlpha);
        
        return imageWithoutAlpha;
    }
}

// 是否需要减少原始图片的大小
+ (BOOL)shouldScaleDownImage:(nonnull UIImage *)image {
    BOOL shouldScaleDown = YES;
    
    CGImageRef sourceImageRef = image.CGImage;
    CGSize sourceResolution = CGSizeZero;
    sourceResolution.width = CGImageGetWidth(sourceImageRef);
    sourceResolution.height = CGImageGetHeight(sourceImageRef);
    // 图片总共像素
    float sourceTotalPixels = sourceResolution.width * sourceResolution.height;
    // 如果图片的总像素大于一定比例，则需要做简化处理
    float imageScale = kDestTotalPixels / sourceTotalPixels;
    if (imageScale < 1) {
        shouldScaleDown = YES;
    } else {
        shouldScaleDown = NO;
    }
    
    return shouldScaleDown;
}

// 获取图片的色彩空间
+ (CGColorSpaceRef)colorSpaceForImageRef:(CGImageRef)imageRef {
    // current
    CGColorSpaceModel imageColorSpaceModel = CGColorSpaceGetModel(CGImageGetColorSpace(imageRef));
    CGColorSpaceRef colorspaceRef = CGImageGetColorSpace(imageRef);
    
    BOOL unsupportedColorSpace = (imageColorSpaceModel == kCGColorSpaceModelUnknown ||
                                  imageColorSpaceModel == kCGColorSpaceModelMonochrome ||
                                  imageColorSpaceModel == kCGColorSpaceModelCMYK ||
                                  imageColorSpaceModel == kCGColorSpaceModelIndexed);
    if (unsupportedColorSpace) {
        colorspaceRef = SDCGColorSpaceGetDeviceRGB();
    }
    return colorspaceRef;
}
```

#### SDWebImageCompat

该类就提供了一个全局方法`SDScaledImageForKey`，这个方法根据原始图片绘制一张放大或者缩小的图片。



## 工具类

1.  NSData + ImageContentType

    用一个类方法 + (SDImageFormat)sd_imageFormatForImageData:(nullable NSData *)data获取图片类型, 具体实现是获取图片data数据的第一个字节数据, 然后枚举出图片类型: JPEG/PNG/GIF/TIFF/WEBP/HEIC

2.   SDWebImageCompat

    该类就提供了一个全局方法`SDScaledImageForKey`，这个方法根据原始图片绘制一张放大或者缩小的图片。

## 代码中值得学习的地方

1.  **缓存读取中的NSOperation**

    这是一个十分巧妙的设计！NSOperation对象并没有管理真实的任务，只使用了它的取消功能，具体任务被分发到串行队列上执行。在业务层做取消操作的时候，操作的是NSOperation，只是标志取消了。在提取任务执行开始，在缓存中的串行队列上调度的block会判断NSOperation的状态，决定是否继续查询缓存。 参看下面代码：

    ```objective-c
    - (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options done:(nullable SDCacheQueryCompletedBlock)doneBlock {
        if (!key) {
         ...
        
        // First check the in-memory cache...
        UIImage *image = [self imageFromMemoryCacheForKey:key];
        BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryDataWhenInMemory));
        if (shouldQueryMemoryOnly) {
            if (doneBlock) {
                doneBlock(image, nil, SDImageCacheTypeMemory);
            }
            return nil;
        }
        
        NSOperation *operation = [NSOperation new];
        void(^queryDiskBlock)(void) =  ^{
            if (operation.isCancelled) {
                // do not call the completion if cancelled
                return;
            }
            
            @autoreleasepool {
                NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
                ...
                if (doneBlock) {
                    if (options & SDImageCacheQueryDiskSync) {
                        doneBlock(diskImage, diskData, cacheType);
                    } else {
                        dispatch_async(dispatch_get_main_queue(), ^{
                            doneBlock(diskImage, diskData, cacheType);
                        });
                    }
                }
            }
        };
        
        if (options & SDImageCacheQueryDiskSync) {
            queryDiskBlock();
        } else {
            dispatch_async(self.ioQueue, queryDiskBlock);
        }
        
        return operation;
    }
    ```



