https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/cross_development/Using/using.html  
这个是苹果的原文链接。这篇文章的中心思想是让你可以安全、有效的处理sdk版本相关的问题，这样可以持续的引入新的api和新的框架，同时保证你的应用可以前向兼容，避免崩溃等问题；同时可以用来适配不同版本的设备和sdk等等。 作为一名进阶开发者，可以说这是一项非常重要而且实用的技术。  
```
if ([UIPrintInteractionController class]) {
    // Create an instance of the class and use it.
} else {
    // Alternate code path to follow when the
    // class is not available.
}
```

1. 判断class  wererere

