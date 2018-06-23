## iOS中的图片使用方式、内存对比和最佳实践

### 预备知识
我们的对比主要关注内存的占用情况。对比的格式是jpg和png这两种最广泛使用的格式，分别代表了有损压缩和无损压缩；关于它们的特点和介绍，可以参考郭耀源的这篇文章：[移动端图片格式调研](https://blog.ibireme.com/2015/11/02/mobile_image_benchmark/)。我们可以看到，在iOS设备上它们的解码消耗在一个量级，速度较快。   
wwdc2018苹果重点关注了图片的内存占用情况（因为此前大家的通用做法实际上是相对低效的），并且给了大家一些指导。这里我们会用demo实验的方式（并且通过工具来进行benchmark），为大家揭示一些事实和现象，供大家去理解和分析，从而去形成我们代码的最佳实践。
### Demo:[DownSampleDemo](https://github.com/dustturtle/DownSampleDemo)

### 场景
首先，苹果告诉我们，图片在应用中主要的内存占用（这通常发生在图片要被载入并显示时）和图片本身的大小实际是无关的；重要的是图片的尺寸。decode buffer的计算方式是width*height\*N,这里的N通常是4(最常见的ARGB888格式)，N取决于你显示所使用的格式。但很多时候，我们会直接把一个UIImage传递给UIImageView， 实际上该View的尺寸可能远远的小于UIImage本身。

### 使用图片的三种方式:

#### 方式A
```
image1.image = UIImage(named: "000.jpg")
```
这是我们最通常使用图片的方式，可能有人会想到imagewithcontentsoffile,实际上它和上面的方法只有一个不同之处： 正常情况下imageNamed会缓存这个图片在整个app存续期间，这样就无需重复的decode；而imagewithcontentsoffile则没有这个缓存，图片不使用了就会释放内存。但这对我们的case并无帮助，因为我们是在使用这些图片，并且观察它们的内存占用.  

#### 方式B

```
+ (UIImage*)OriginImage:(UIImage *)image scaleToSize:(CGSize)size
{
    // 创建一个bitmap的context
    // 并把它设置成为当前正在使用的context
    UIGraphicsBeginImageContext(size);
    
    // 绘制改变大小的图片
    [image drawInRect:CGRectMake(0, 0, size.width, size.height)];
    
    // 从当前context中创建一个改变大小后的图片
    UIImage* scaledImage = UIGraphicsGetImageFromCurrentImageContext();
    
    // 使当前的context出堆栈
    UIGraphicsEndImageContext();
    
    // 返回新的改变大小后的图片
    return scaledImage;
}
```
这是一种被广泛使用的缩放图片的办法。

#### 方式C
```
    func downsample(imageAt imageURL: URL, to pointSize: CGSize, scale: CGFloat) -> UIImage
    {
        let sourceOpt = [kCGImageSourceShouldCache : false] as CFDictionary
        // 其他场景可以用createwithdata (data并未decode,所占内存没那么大),
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
这是苹果介绍给我们的新方法。

#### 关于测试
测试设备： 1.iphone8 11.4  2.iphone6 12beta2  
测试图片格式： png/jpg  
测试的图片使用方式：3种  
我们将2560*1440大小的图片放到282\*138的view中，对于缩放我们使用2x的格式来显示以保证其效果。

实验数据的细节在DownSampleDemo的ViewController.swift的注释中；直接告诉大家结论：   
1. xcode面板上所显示的内存占用并不可靠，在很多case下应用占用的内存要比其上显示的多的多（特别是对于iOS11.4的设备）。但可以用来粗略的查看内存占用，如果这里超了很多，那肯定是超了  
2. 可以使用instruments的allocation和memory debugging的memgraph+命令行分析这两种方式来观测内存的占用   
3. 第一种方式下jpg载入后的内存分为imageIO和IOkit两部分，png载入只有imageIO部分，jpg内存占用比png要少不少  
4. 第二种方式的数据在xcode面板上失真，通过另外两种方式可以看到它非常不靠谱。内存消耗甚至可能超过第一种。  
5. 第三种方式的内存占用非常完美，严格的遵守了公式计算出来的大小（跟你downsample后的尺寸有关），内存占用在CG raster data中。    
6. **memgraph命令： vmmap --summary xxx.memgraph**  
7. **allocation我们只需要看VM allocation栏目下的dirty size和swapped size，这里需要手动的打开snapshot才能看到该栏目的数据,并且需要适当的重复多次运行才能保证结果的正确性**

VMTracker运行截图：
![](http://7xr2v8.com1.z0.glb.clouddn.com/vmtracker_dirtySize_564.png)
memgraph命令结果截图：
![](http://7xr2v8.com1.z0.glb.clouddn.com/memgraph_564.png)

### 最后，来自苹果的最佳实践:强烈推荐大家使用downsample来处理大图！Thank you!:]

