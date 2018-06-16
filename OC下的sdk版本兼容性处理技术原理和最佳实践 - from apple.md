## 前言
这里的主题是兼容性，也可以称之为availablity。
实际上，在swift里面推荐的最佳实践就是称作是available的语言内置技术，语法是@available (大概是这样)。而在oc下，情况要复杂一些，这些细节大部分都和OC的runtime有关。我们这篇文章是希望给OC的使用者一个正确指导。

##正文

https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/cross_development/Using/using.html  
这个是苹果的原文链接。我们希望让你可以安全、有效的处理sdk版本相关的问题，这样可以持续的引入新的api和新的框架（需要结合弱引用技术），同时保证你的应用可以前向兼容，避免崩溃等问题；同时可以用来适配不同版本的设备和sdk、发现deprecated的api调用。 作为一名进阶开发者，可以说这是一项非常重要而且实用的技术。 
###details
###1. 判断class是否可用
```
if ([UIPrintInteractionController class]) {
    // Create an instance of the class and use it.
} else {
    // Alternate code path to follow when the
    // class is not available.
}
```
需要注意的是这个方法在很老的编译器/baseSDK/target版本下不起作用(但一般来说我们的项目不会受这个限制，只是给大家了解一下)，需要使用下面的：
```we
Class cls = NSClassFromString (@"NSRegularExpression");
if (cls) {
    // Create an instance of the class and use it.
} else {
    // Alternate code path to follow when the
    // class is not available.
}
```

###2. 判断oc的方法是否可用
```
if ([UIImagePickerController instancesRespondToSelector:
              @selector (availableCaptureModesForCameraDevice:)]) {
    // Method is available for use.
    // Your code can check if video capture is available and,
    // if it is, offer that option.
} else {
    // Method is not available.
    // Alternate code to use only still image capture.
}
```
###3. 判断c函数是否可用
```
if (CGColorCreateGenericCMYK != NULL) {
    CGColorCreateGenericCMYK (0.1,0.5.0.0,1.0,0.1);
} else {
    // Function is not available.
    // Alternate code to create a color object with earlier technology
}
```
###4. 关于弱引用库
当你要使用的库不是你所有的target都支持的时候，你需要弱引用之。在库的选项里面将其从required改为optional即可。  
###5. 对不同版本的sdk进行条件编译

```
#ifdef __MAC_OS_X_VERSION_MAX_ALLOWED
    // code only compiled when targeting OS X and not iOS
    // note use of 1050 instead of __MAC_10_5
#if __MAC_OS_X_VERSION_MAX_ALLOWED >= 1050
    if (CGColorCreateGenericCMYK != NULL) {
        CGColorCreateGenericCMYK(0.1,0.5.0.0,1.0,0.1);
    } else {
#endif
    // code to create a color object with earlier technology
#if __MAC_OS_X_VERSION_MAX_ALLOWED >= 1050
    }
#endif
#endif
}
```

###6. 代码运行时判断操作系统/库的版本号
####a. 操作系统
```
NSString *osVersion = [[UIDevice currentDevice] systemVersion];
```
####b.库
mac上的appkit,像这样：

```
APPKIT_EXTERN double NSAppKitVersionNumber;
#define NSAppKitVersionNumber10_0 577
#define NSAppKitVersionNumber10_1 620
#define NSAppKitVersionNumber10_2 663
#define NSAppKitVersionNumber10_2_3 663.6
#define NSAppKitVersionNumber10_3 743
```
判断代码示例：

```
if (floor(NSAppKitVersionNumber) <= NSAppKitVersionNumber10_0) {
  /* On a 10.0.x or earlier system */
} else if (floor(NSAppKitVersionNumber) <= NSAppKitVersionNumber10_1) {
  /* On a 10.1 - 10.1.x system */
}
```
iOS上常见的是 NSFoundationVersionNumber， 在purelayout这个框架的代码里可以看到如何使用。

That's all, thank you.

