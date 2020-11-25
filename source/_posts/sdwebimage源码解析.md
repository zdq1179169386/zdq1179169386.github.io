---
layout: 
title: SDWebimage 源码解析
date: 2018-12-19 09:55:19
tags: [SDWebimage 源码]

---

![](/images/sdwebimage.jpg)


SDWebimage 是iOS 开发的图片加载库，最主要的类的就是 `SDWebImageManager` ,它拥有 `SDImageCache` 类型的 `imageCache` （图片缓存） 和 `SDWebImageDownloader` 类型的 `imageDownloader` (图片下载器)。 

<!--more-->

### SDWebImageManager

当根据 URL 去加载图片时，会先去判断是否有内存缓存，有： 直接去显示图片，没有：再去判断是否有磁盘缓存，有：将磁盘缓存加载到内存缓存中，然后再去显示图片，没有：执行下载操作。

### SDWebImageDownloader

`SDWebImageDownloader` 是负责管理下载操作的，会将每个下载请求封装成operation ，然后加入到 `downloadQueue`  并发队列中（最大并发数是6），然后会记录这次下载请求，就是加这次operation 添加到 `URLOperations` 中， `URLOperations` 是个字典，每次下载前会根据URL 判断是否有对应的operation，如果有就不重复添加。成功之后，会从 `URLOperations` 删除对应的 operation。

下载成功之后，会先将图片数据存到磁盘中，然后加载一份到内存缓存中，然后再去显示图片。

### SDImageCache

```
@interface SDImageCache ()

#pragma mark - Properties
@property (strong, nonatomic, nonnull) NSCache *memCache;
@property (strong, nonatomic, nonnull) NSString *diskCachePath;
@property (strong, nonatomic, nullable) NSMutableArray<NSString *> *customPaths;
@property (SDDispatchQueueSetterSementics, nonatomic, nullable) dispatch_queue_t ioQueue;

@end
```

SDImageCache 是负责管理图片缓存的。

4.0 版本的内存缓存是拥有一个 `NSCache` 类型的memCache 对象，`NSCache` 的好处是当收到内存警告时，它会自动移除对象来释放内存。磁盘缓存的部分 ，是使用 `NSFileManager`  进行文件存储的。还有一个串行的 ioQueue ，将图片数据写入到磁盘中，或者从磁盘中取图片数据，都是在这个串行队列中进行。

```
@interface SDImageCache ()

#pragma mark - Properties
@property (nonatomic, strong, readwrite, nonnull) id<SDMemoryCache> memoryCache;
@property (nonatomic, strong, readwrite, nonnull) id<SDDiskCache> diskCache;
@property (nonatomic, copy, readwrite, nonnull) SDImageCacheConfig *config;
@property (nonatomic, copy, readwrite, nonnull) NSString *diskCachePath;
@property (nonatomic, strong, nullable) dispatch_queue_t ioQueue;

@end
```

5.0之后，它包含一个`SDMemoryCache `类型的内存缓存对象`memoryCache`，和`SDDiskCache `类型的磁盘缓存对象`diskCache`。IO 读写队列还是没变

### SDMemoryCache

首先 `SDMemoryCache` 继承了`NSCache`，它拥有个`NSMapTable` 类型的weakCache属性。

NSMapTable 的特点如下：

> NSMapTable和NSDictionary相对应，相对于NSDictionary/NSMutableDictionary，NSMapTable有如下的特征：
NSDictionary/NSMutableDictionary会copy对应的key，强引用相应的value。
NSMapTable是可变的，没有一个不变的类与其对应。
NSMapTable可以对其key和value弱引用，在这种情况下当key或者value被释放的时候，此entry会自动从NSMapTable中移除。
NSMapTable在加入一个（key，value）的时候，可以对其value设置为copy。
NSMapTable可以包含任意指针，使用指针去做相等或者hashing检查。

[参考](https://juejin.im/post/5a321cba6fb9a0450671a42c)

```
self.weakCache = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
```
`NSPointerFunctionsWeakMemory ` 使用 weak 储存 value 值

另外它还拥有一个 `weakCacheLock` 锁对象，在读写的时候加锁，防止多线程出错
```
@property (nonatomic, strong, nonnull) dispatch_semaphore_t weakCacheLock
```

另外，它还自己监听了 `UIApplicationDidReceiveMemoryWarningNotification` 通知，收到内存警告的时候，会释放内存。(不太清楚这里为啥还要监听内存警告通知，不是说nscache 会自动移除对象来释放内存吗)

疑问？
> SDMemoryCache 本身是继承 NSCache 的，为啥还要用 weakCache 来存储数据呢？况且之前的版本，SDImageCache 就有一个 NSCache 类型的属性 memoryCache，后续版本为啥要这么修改呢？

原因：
> NSCache 本身是会在收到内存警告的时候，会自动释放一部分内存的，假如 imageview 需要一张图，这时候会先去内存找这张图，没有再去磁盘中读取，加载一份到内存中，同时设置到 imageView 上，如果这时，收到内存警告了，memoryCache 会自动释放掉一部分内存，刚好就是刚刚读取到内存中的这张图片，如果是列表反复滑动的时候，就会出现频繁去磁盘中读取图片的情况，我们都知道去磁盘中读取数据是非常耗时的，所以 Sd 使用了  NSMapTable类型的 weakCache 。自己实现了收到内存警告的方法，修改删除逻辑，优先删除老的数据，而不是最近使用的数据。

### SDDiskCache

它拥有一个 `fileManager` 对象，负责图片存储和删除。

1， `SDImageCache`初始化的时候，会调用`migrateDiskCacheDirectory` 方法，做一次数据迁移


```
- (void)migrateDiskCacheDirectory {
    if ([self.diskCache isKindOfClass:[SDDiskCache class]]) {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            // ~/Library/Caches/com.hackemist.SDImageCache/default/
            NSString *newDefaultPath = [[[self userCacheDirectory] stringByAppendingPathComponent:@"com.hackemist.SDImageCache"] stringByAppendingPathComponent:@"default"];
            // ~/Library/Caches/default/com.hackemist.SDWebImageCache.default/
            NSString *oldDefaultPath = [[[self userCacheDirectory] stringByAppendingPathComponent:@"default"] stringByAppendingPathComponent:@"com.hackemist.SDWebImageCache.default"];
            dispatch_async(self.ioQueue, ^{
                [((SDDiskCache *)self.diskCache) moveCacheDirectoryFromPath:oldDefaultPath toPath:newDefaultPath];
            });
        });
    }
}
```

2, `SDImageCache ` 会监听APP 进入后台通知，当APP 进入后台的时候，会调用`diskCache ` 去删除过期数据

```
- (void)deleteOldFilesWithCompletionBlock:(nullable SDWebImageNoParamsBlock)completionBlock {
    dispatch_async(self.ioQueue, ^{
        [self.diskCache removeExpiredData];
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}
```
过期时间是一周，删除的话，会先降到存储size 最大值的一半。

### 缓存查找

会先从内存缓存中查找 `imageFromMemoryCacheForKey`, 内存中不存在，才会从磁盘缓存查找`diskImageDataBySearchingAllPathsForKey` 

从缓存中查找图片数据，被封装成每次返回一个operation  的方法了, 每次开始前会先判断 这次 operation 是否被取消了，然后会将这次磁盘缓存查找，提交串行队列 `ioQueue` 中去执行

```
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key done:(nullable SDCacheQueryCompletedBlock)doneBlock {
    if (!key) {
        if (doneBlock) {
            doneBlock(nil, nil, SDImageCacheTypeNone);
        }
        return nil;
    }
    // First check the in-memory cache...
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        NSData *diskData = nil;
        if ([image isGIF]) {
            diskData = [self diskImageDataBySearchingAllPathsForKey:key];
        }
        if (doneBlock) {
            doneBlock(image, diskData, SDImageCacheTypeMemory);
        }
        return nil;
    }

    NSOperation *operation = [NSOperation new];
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) {
            // do not call the completion if cancelled
            return;
        }

        @autoreleasepool {
            NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage && self.config.shouldCacheImagesInMemory) {
                NSUInteger cost = SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            if (doneBlock) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    doneBlock(diskImage, diskData, SDImageCacheTypeDisk);
                });
            }
        }
    });
    return operation;
}
```

### 解码

为什么要解码 ?

>事实上，不管是 JPEG 还是 PNG 图片，都是一种压缩的位图图形格式。只不过 PNG 图片是无损压缩，并且支持 alpha 通道，而 JPEG 图片则是有损压缩，可以指定 0-100% 的压缩比,因此，在将磁盘中的图片渲染到屏幕之前，必须先要得到图片的原始像素数据，才能执行后续的绘制操作，这就是为什么需要对图片解压缩的原因。

`SDWebImage ` 的流程： 当根据 URL 下载一张图片时，成功了之后，这时在在下载成功的回调中我们得到的是 imageData 对象， 然后我们会转成 image ,这时的image 可能是 jpg 或者 png 格式的，然后我们会将图片渲染出来，也就是设置到 imageView 的 image 属性上，这时系统会先将图片解压得到原始数据，也就是位图数据，然后再进行绘制。将压缩的图片数据解码成未压缩的位图数据，是一个非常长耗时的CPU操作，而且是发生在主线程。所以这对我们的APP性能是很严重的影响。`SDWebImage ` 将 image 的缓存查找提交到了 `ioQueue` 串行队列，异步执行，会创建一个分线程按顺序执行， 所以图片的解码也会在这条线程上执行，从而不会阻塞主线程，

经过前面的分析，我们知道 `SDImageCache` 拥有一个 `ioQueue` 队列，会将图片的写入磁盘操作和从磁盘中读取数据操作（包括图片解码），提交到这个队列上，异步执行会创建一条新的线程按顺序执行任务。

同时从磁盘中读取到图片数据之后，会加载一份到内存缓存中，所以内存缓存中持有的也是解码之后的图片数据。

解码方式:

1、 使用UIGraphic创建Context，并调用UIImage的drawInRect函数。这个方法虽然是UI开头的，但是确实线程安全的

2、 创建带Alpha的CGContext，然后把CGImage绘制上去

3、 创建不带Alpha的CGContext，然后把CGImage绘制上去

4、 ImageIO创建UIImage有个选项shouldCacheImmediately，设为true之后创建UIImage同时就会进行解码，kCGImageSourceShouldCache : 指定是否应以解码形式缓存图像，64 位，默认是开启的

SDWebImage 用的就是 `Core Graphics `,  4.0版本的解码部分代码，主要在 `SDWebImageDecoder` 这个类里

```

+ (nullable UIImage *)decodedImageWithImage:(nullable UIImage *)image {
if (![UIImage shouldDecodeImage:image]) {
    return image;
}
    
// autorelease the bitmap context and all vars to help system to free memory when there are memory warning.
// on iOS7, do not forget to call [[SDImageCache sharedImageCache] clearMemory];
@autoreleasepool{
    
    CGImageRef imageRef = image.CGImage;
    CGColorSpaceRef colorspaceRef = [UIImage colorSpaceForImageRef:imageRef];
    
    size_t width = CGImageGetWidth(imageRef);
    size_t height = CGImageGetHeight(imageRef);
    size_t bytesPerRow = kBytesPerPixel * width;

    // kCGImageAlphaNone is not supported in CGBitmapContextCreate.
    // Since the original image here has no alpha info, use kCGImageAlphaNoneSkipLast
    // to create bitmap graphics contexts without alpha info.
    CGContextRef context = CGBitmapContextCreate(NULL,
                                                 width,
                                                 height,
                                                 kBitsPerComponent,
                                                 bytesPerRow,
                                                 colorspaceRef,
                                                 kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
    if (context == NULL) {
        return image;
    }
    
    // Draw the image into the context and retrieve the new bitmap image without alpha
    CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
    CGImageRef imageRefWithoutAlpha = CGBitmapContextCreateImage(context);
    UIImage *imageWithoutAlpha = [UIImage imageWithCGImage:imageRefWithoutAlpha
                                                     scale:image.scale
                                               orientation:image.imageOrientation];
    
    CGContextRelease(context);
    CGImageRelease(imageRefWithoutAlpha);
    
    return imageWithoutAlpha;
}
}
```

5.8 版本的解码部分在`SDImageCoderHelper` 这个类里面，代码大致相同。

Image IO 的解码方式

```
//ImageIO
- (UIImage *)resizedImage:(NSURL *)url size:(CGSize)size{
    CFURLRef urlRef = CFBridgingRetain(url);
    CFDictionaryRef imageOptions;
    CFStringRef imageKeys[2];
    CFTypeRef imageValues[2];
    
    imageKeys[0] = kCGImageSourceShouldCache;
    imageValues[0] = (CFTypeRef)kCFBooleanTrue;
    imageKeys[1] = kCGImageSourceShouldAllowFloat;
    imageValues[1] = (CFTypeRef)kCFBooleanTrue;
    
    imageOptions = CFDictionaryCreate(NULL, (const void **)imageKeys, (const void**)imageValues, 2, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
    
    CGImageSourceRef  imageSource = CGImageSourceCreateWithURL(urlRef, imageOptions);
    CFRelease(imageOptions);
    if (imageSource == NULL) {
        return NULL;
    }
    CGImageRef  imageRef = CGImageSourceCreateImageAtIndex(imageSource, 0, NULL);
    UIImage * image = [[UIImage alloc] initWithCGImage:imageRef];
    return image;
}
```



### 高清图处理

在解码大图的时候，用到了 `CGBitmapContextCreate` 函数，它会生成一块大小 width * height * 4 的内存。所以大图加载的时候，出现内存陡增的情况。

解决办法：

1，[一个是创建缩略图](https://juejin.im/post/5d70c3616fb9a06afe12b762)

2，[一个是分块绘制](https://juejin.im/post/5adde71c6fb9a07aa63163eb)



### 如何优化 `SDWebImage`

1， 缓存命中
可以采用 LRU 算法，最近最少算法，如果数据最近被访问过，那么将来被访问的几率也更高，如果一个图片被访问了，同时将它挪到数组的前端，那下次查找速度也会快很多，当缓存超过限制了，将优先数组尾部的数据，因为那些是比较少被访问到的。

2，图片解码

`Core Graphics` 和 `Image I/O` 都可以进行解码操作，但是 sd 为啥要选用  `Core Graphics` ，不是很清楚，因为从网上资料看， Image IO 的性能优于 Core Graphics。
源码上，我好像只看到解码动图部分用了，Image IO 解码的 那些参数。 

参考:

[iOS平台图片编解码入门教程（Image/IO篇）](https://dreampiggy.com/2017/10/30/iOS%E5%B9%B3%E5%8F%B0%E5%9B%BE%E7%89%87%E7%BC%96%E8%A7%A3%E7%A0%81%E5%85%A5%E9%97%A8%E6%95%99%E7%A8%8B%EF%BC%88Image:IO%E7%AF%87%EF%BC%89/)

[如何打造易扩展的高性能图片组件](https://zhuanlan.zhihu.com/p/26955368)

[如何避免图像解压缩的时间开销](https://longtimenoc.com/archives/ios%E5%A6%82%E4%BD%95%E9%81%BF%E5%85%8D%E5%9B%BE%E5%83%8F%E8%A7%A3%E5%8E%8B%E7%BC%A9%E7%9A%84%E6%97%B6%E9%97%B4%E5%BC%80%E9%94%80)

[图像调整大小技术](https://nshipster.com/image-resizing/)

[iOS Rendering 渲染全解析](https://juejin.im/post/5ec35cc55188256d92438174#heading-8)

[CGImageRef](https://my.oschina.net/u/2340880/blog/406437)

[ImageIO](https://my.oschina.net/u/2340880/blog/838680)

[Image IO 学习笔记](https://www.jianshu.com/p/4dcd6e4bdbf0)

源码：

[ReadSourceCode](https://github.com/zdq1179169386/ReadSourceCode)