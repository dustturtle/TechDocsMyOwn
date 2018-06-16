###一些概念
backing store / UIGraphicsImageRenderer （iOS 10+） / PrefetchDataSource (iOS 10+)

iOS12中系统会对一些场景下的内存使用进行优化（主要的case是这样：根据灰度图/彩色图这样的色域宽度的不同分配合适大小的内存，而在之前则分配一视同仁的内存，造成了浪费），但需要注意我们的代码如果使用的不恰当，苹果可能会无法将优化应用到我们的app上面。

  
另外，我们需要知道的一件事情是cpu和内存的占用实际上是相互影响的。其原因的细节有些复杂，但简单来说可以这样理解：cpu和内存作为两种最关键的资源，我们可以通过技术手段进行trade off。苹果系统也大量的应用了此类技术。因此，你可能会发现，当你内存占用过高时，cpu可能也会卡顿，反之也有可能成立。

###details
1. 如何正确的使用UIImage,以及在tableview/collectionview中的使用姿势  

   ```
   func downsample(imageAt imageURL: URL, to pointSize: CGSize, scale: CGFloat) -> UIImage
    {
        let sourceOpt = [kCGImageSourceShouldCache : false] as CFDictionary
        // 其他场景可以用createwithdata,
        let source = CGImageSourceCreateWithURL(imageURL as CFURL, sourceOpt)!
        let maxDimension = max(pointSize.width, pointSize.height) * scale
        let downsampleOpt = [kCGImageSourceCreateThumbnailFromImageAlways : true,
                             kCGImageSourceShouldCacheImmediately : true ,
                             kCGImageSourceCreateThumbnailWithTransform : true,
                             kCGImageSourceThumbnailMaxPixelSize : maxDimension] as CFDictionary
        let downsampleImage = CGImageSourceCreateThumbnailAtIndex(source, 0, downsampleOpt)!
        return UIImage(cgImage: downsampleImage)
    }
    
   ```
   我们这里引入了downsample技术来处理图片，该操作已经包含了decode(cpu expensive)。  
   
   痛点解析：
   从UIImage-> UIImageView 我们创建了data buffer 和 decode buffer 然后才能同步到 frame buffer；这里的data buffer 和 decode buffer会一直存在，且其占用的内存大小只跟UIImage的尺寸有关，而非UIImageView实际用到的。也就是说，你如果是一个的imageview，使用了一张大图填充，那么会浪费非常多的内存；采用常规的手段对image进行缩放实际上也不能解决问题，decode buffer从载入image开始就会存在。所以注意kCGImageSourceShouldCache和kCGImageSourceShouldCacheImmediately，它们起了很大的作用.
     
 而在tableview这样的地方使用图片，仅仅使用downsample技术（在获取cell的地方）是不够的。你会发现在上下滑动时出现卡顿。究其原因是downsample消耗了cpu资源，而frame buffer的显示也需要cpu，这样就会引起掉帧；卡顿的情况也会加大电量消耗（cpu短期负载高）。  
 
 我们有两个技术来应对：  
 a. PrefetchDataSource 它可以让我们在cell显示之前就提前准备好，避免了冲突。我们这里expensive的操作就是准备downsample的image，将它放到该方法中执行，然后传递给数据源.  
 b. 让耗时操作在后台运行，避免卡顿界面。需要注意这里得避免线程爆炸：我们应该创建1个共享的serialQueue来统一执行这类操作。否则会造成cpu线程频繁切换，代价非常高。

2. 尽量使用image assets来放置图片。苹果有一箩筐的优化技术应用在上面，使得其加载速度、内存消耗以及功能上都非常的优秀，相对传统的方法（放在bundle之类）。
3. image assets里面有个preserve vector data，它在你需要缩放的时候表现非常棒，但你应该仅在此场景下勾选。它很贵。如果仅仅需要两个不同分辨率的图，建议你多费点事，再准备一套放到assets中。
4. 我们有时候可能会希望通过重写draw方法实现特别的页面。我们告诉你，尽量不要这么做。对image或者文字直接在里面draw会产生大量的backup store, 而如果你通过subview的方式把imageview和label放置到你的页面里，则会大大减少backup store，这玩意会占用大块内存。   
另外，尽量使用calayer.cornerradius,mask相关的api需要额外的image buffer。设置backgroundcolor来达到目的。（总之，尽量使用高级的api，并且组合控件，这是我个人的观点。）
5. 使用UIGraphicsImageRenderer而不是UIGraphicsBeginImageContextWithOptions这样的传统api。UIGraphicsBeginImageContext很可能会导致iOS12上的优化不起效，因为你的设置是固定的。

###TODO: demo needed.