# iOS内存优化与监控

有一种情况奔溃在开发期暴露不了，但是用户在使用时确实因为内存溢出致使app crash 。并且内存溢出的crash时，是不会有什么堆栈信息的，像umeng，透视宝等工具是无法清晰的捕捉到这些奔溃的，这时候就需要我们使用一些其他的办法来监控OOM。


先了解一下OOM的流程，以及如何捕捉OOM
![1](https://scontent-nrt1-1.xx.fbcdn.net/v/t39.2365-6/11891368_135486233460877_1344297194_n.jpg?oh=5b443aacc1f3b4ef5e971841667ba4e3&oe=5B331E7F)

正常的启动流程

![4](https://scontent-nrt1-1.xx.fbcdn.net/v/t39.2365-6/11891347_1679275488959164_500068205_n.jpg?oh=cc33b4088e6a25965185534bad243b7c&oe=5B4AEA29)

如何去处理这些问题

app 需要去重启可能有以下几种原因：

1. app升级
2. app主动调用exit或abort方法
3. app crashed
4. 用户强退
5. 设备重启，
6. app 内存溢出，包括前台溢出（FOOM）后台溢出 （BOOM）

下面是facebook的分析流程

![666](https://scontent-nrt1-1.xx.fbcdn.net/v/t39.2365-6/11891371_510065462479783_784800705_n.jpg?oh=2983e68f166364bc15a737c42a53c554&oe=5B4BA651)

a) App没有升级；

b) App没有调用exit()或abort()退出；

c) App没有出现crash；

d) 用户没有强退App；

e) 系统没有升级/重启；

f) App当时没有后台运行；

g) App出现FOOM。

推荐使用腾讯的OOMDetector组件，具体原理看下面这篇文章

[iOS爆内存问题解决方案-OOMDetector组件](https://segmentfault.com/a/1190000012825286)

**开发期我们利用工具和代码审查可以去优化的地方**

优化：

1） UIGraphicsEndImageContext：

UIGraphicsBeginImageContext和UIGraphicsEndImageContext必须成双出现，不然会造成context泄漏。

2）UIWebView：

无论是打开网页，还是执行一段简单的js代码，UIWebView都会占用APP大量内存。而WKWebView不仅有出色的渲染性能，而且它有自己独立进程，一些网页相关的内存消耗移到自身进程里，最适合取替UIWebView。

3）autoreleasepool：

通常autoreleased对象是在runloop结束时才释放。如果在循环里产生大量autoreleased对象，内存峰值会猛涨，甚至出现OOM。适当的添加autoreleasepool能及时释放内存，降低峰值。

4）互相引用：

比较容易出现互相引用的地方是block里使用了self，而self又持有这个block，只能通过代码规范来避免。另外NSTimer的target、CAAnimation的delegate，是对Obj强引用。目前微信通过自己实现的MMNoRetainTimer和MMDelegateCenter来规避这类问题。**其实实现MMNoRetainTimer等类，就是让 类.delegate = 类  用自身的互相引用，代替互相引用**

5）大图片处理：

举个例子，以往图片缩放接口是这样写的：
![1500839-1811a6f01f3e457c.](https://upload-images.jianshu.io/upload_images/1500839-1811a6f01f3e457c..jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)


但处理大分辨率图片时，往往容易出现OOM，原因是-[UIImage drawInRect:]在绘制时，先解码图片，再生成原始分辨率大小的bitmap，这是很耗内存的。解决方法是使用更低层的ImageIO接口，避免中间bitmap产生：
![1500839-f2c17edbcb6ff49a.](https://upload-images.jianshu.io/upload_images/1500839-f2c17edbcb6ff49a..jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

6）大视图：

大视图是指View的size过大，自身包含要渲染的内容。超长文本是微信里常见的炸群消息，通常几千甚至几万行。如果把它绘制到同一个View里，那将会消耗大量内存，同时造成严重卡顿。最好做法是把文本划分成多个View绘制，利用TableView的复用机制，减少不必要的渲染和内存占用。

7） 频繁的创建和销毁对象对内存有很大的开销，也就是创建很多临时对象，比如数组的赋值什么的，很多人喜欢创建临时数组去保存东西，一直保存一个对象内存占用更少.

工具：

1. XCode的Analyze。
2. instrument 工具
3. facebook 开源的
   (1) 静态代码分析工具 [Infer](https://www.oschina.net/p/infer)
   (2) iOS内存监测工具 [FBMemoryProfiler](https://www.oschina.net/p/fbmemoryprofiler) 
   (3) 使用介绍[FBRetainCycleDetector](https://juejin.im/entry/59a61fd4f265da249600daeb)
4. 微信 [MLeaksFiner] (https://wereadteam.github.io/2016/02/22/MLeaksFinder/)

####参考链接
[Reducing FOOMs in the Facebook iOS app](https://code.facebook.com/posts/1146930688654547/reducing-fooms-in-the-facebook-ios-app/)
[微信团队原创分享：iOS版微信的内存监控系统技术实践](http://www.cocoachina.com/ios/20180305/22458.html)




