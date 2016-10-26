---
layout: post
title: "webp调研"
date: 2016-01-31 16:04:07 +0800
comments: true
categories: Android iOS 图片优化 Webp 
---

#WebP简介：
&emsp;&emsp;WebP 格式是 Google 于2010年发布的一种支持有损压缩和无损压缩的图片文件格式，派生自图像编码格式 VP8。它具有较优的图像数据压缩算法，能带来更小的图片体积，而且拥有肉眼识别无差异的图像质量，同时具备了无损和有损的压缩模式、Alpha 透明以及动画的特性。目前googleG+、YouTube以及Google Play全站都在使用WebP格式的图片。[15年双11手淘前端技术巡演中提到](https://github.com/amfe/article/issues/21)，淘宝3年前native层面就已经支持WebP， 两年前H5页面全面支持。QQ空间装扮、淘宝广告图，[Facebook Android](https://code.facebook.com/posts/485459238254631/improving-facebook-on-android/)也都使用WebP。

以下通过研究WebP图片格式，尽可能全面地了解WebP图片的优劣势以及应用WebP图片给我们带来的收益以及风险，最终达到提升用户体验，提升图片加载速度，节省带宽的目的。

-----------

#WebP优势
##官方测试：
[根据测试](https://developers.google.com/speed/webp/docs/webp_lossless_alpha_study)，WebP无损压缩的图片比PNG格式图片，文件大小上少 26%；

[根据测试](https://developers.google.com/speed/webp/docs/webp_study)，WebP有损图片在同样[SSIM](https://en.wikipedia.org/wiki/Structural_similarity)质量指标上比JPEG格式图片少25~34%。 SSIM是一种衡量两张数字影像相似的指标

###有损压缩测试方法简述：

1.将PNG图片设置不同的压缩参数压缩成JPEG图片，记录压缩后的对比的SSIM。

2.将同一张PNG图片压缩成WebP图片，压缩的WebP图片的SSIM指标必须比1中记录的SSIM高。

####jpg、webp相同ssim测试
{% img /images/webp_image/webp-1.jpg %} 

##其他测试：
同样质量的WebP与JPG图片，[在线加载速度测试](http://labs.qiang.it/wen/webp/test.html)。测试的JPG和WebP图片大小如下：

####在线测试图片大小
{% img /images/webp_image/webp-2.jpg %} 


####测试数据折线图如下：
{% img /images/webp_image/webp-3.jpg %} 

从折线图可以看到，WebP虽然会增加额外的解码时间，但由于减少了文件体积，缩短了加载的时间，页面的渲染速度加快了。同时，随着图片数量的增多，WebP页面加载的速度相对JPG页面增快了。所以，使用WebP基本没有技术阻碍，还能带来性能提升以及带宽节省。

通过以上两组对比可以得知，WebP在文件大小以及传输速度上肯定是拥有优势的，将极大节省用户及cdn流量。

#动图

GIF 图片主要应用于图片分享类应用中，如微博等。与传统的 GIF 图比较，动态 WebP 的优势在于：
1.支持有损和无损压缩，并且可以合并有损和无损图片帧；

2.体积更小，GIF 转成有损动态 WebP 后可以减小 64% 的体积，转成无损可以节省 19% 的体积；

3.颜色更丰富，支持 24-bit 的 RGB 颜色以及 8-bit 的 Alpha 透明通道（而GIF 只支持8-bit RGB 颜色以及 1-bit 的透明）；

4.添加了关键帧、metadata 等数据；

##动图测试：

[GIF动图](http://7xscia.com1.z0.glb.clouddn.com/test_001.gif)1 63.9KB   [WebP动图1](http://7xscia.com1.z0.glb.clouddn.com/test_001.gif?imageMogr2/format/webp) 28.9KB

[GIF动图2](http://7xscia.com1.z0.glb.clouddn.com/test_002.gif) 1.7MB   [WebP动图2](http://7xscia.com1.z0.glb.clouddn.com/test_002.gif?imageMogr2/format/webp) 479KB

[GIF动图3](http://7xscia.com1.z0.glb.clouddn.com/test_003.gif) 2MB [WebP动图3](http://7xscia.com1.z0.glb.clouddn.com/test_003.gif?imageMogr2/format/webp) 208KB

#WebP劣势：

1.各个端支持情况不一。这点会在下一节中详细说明。

2.迁移成本较大，需要对所有图片重新编码，考虑到对旧版的支持，需要额外开辟空间存两种格式的图片。

3.编解码速度上，根据Google的测试，目前WebP与JPG相比较，毫秒级别上，编码速度慢10倍，解码速度慢1.5倍。编码速度即可被没影响，我们只是在上传时生成一份WebP图片。解码速度则需要客户端综合节省下的流量来综合考虑。总之带宽节省比cpu消耗更有价值

4.尽管有不少app在使用WebP图片，但与JPG/PNG相比还是太少了，接受度并没有太高。

5.app中部分“归档化”的交互操作，比如图片保存，因此这些交互操作上需要进行WebP编解码成JPG。直接保存下来的webp图片，非常不方便reuse以及review。

#WebP各端支持情况：

####浏览器支持情况：
{% img /images/webp_image/webp-4.jpg %} 

根据对目前国内浏览器占比与 WebP 的兼容性分析，大约有 50% 以上的国内用户可以直接体验到 WebP。

##如何检测浏览器是否支持：
1.JavaScript 能力检测，对支持 WebP 的用户输出 WebP 图片

2.使用 WebP 支持插件：WebPJS

3.有一部分CDN厂商是提供webp检测服务

4.Http header accept type 返回接受的image typ

5.GooglePageSpeed提供自动将jpg转化成webp，提供给支持webp的浏览器上。

##Android：
Android 4.0及以上原生支持； 4.0以下可以使用官方提供提供的[编解码库](https://github.com/alexey-pelykh/webp-android-backport)。

##iOS：
iOS native 不支持，safari目前也不支持。据说ios 10的safari有可能会支持。native支持层面，[google官方](https://github.com/carsonmcdonald/WebP-iOS-example)，以及[第三方](https://github.com/seanooi/iOS-WebP)都提供了解决方案。国内也有不少ios团队在做webp图片的支持工作。

##视觉设计：
Photoshop 原生是不支持WebP的，但有插件提供WebP支持。

####webp各端支持情况
{% img /images/webp_image/webp-5.jpg %} 

#总结：
1.图片占据cdn服务中很大的一部分流量，更节省的图片流量对产品及用户肯定有巨大提升。一部分cnd厂商是支持webp转化服务或者web支持探测服务的，这点可以问我们的cdn厂商。

2.目前各端从jpg/png图片迁移肯定需要一段比较长的过程，并且需要cdn、后台等准备工作。

#参考：
1. WebP官方文档https://developers.google.com/speed/webp/

2. 淘宝前端优化https://github.com/amfe/article/issues/21

3. 腾讯WebP探寻https://isux.tencent.com/introduction-of-webp.html

4. iOS WebP实践https://segmentfault.com/a/1190000006266276

5. 七牛云存储WebP支持https://segmentfault.com/a/1190000002726138

6. 探究WebP一些事儿http://web.jobbole.com/87103/

7. Frequently Asked Questionshttps://developers.google.com/speed/webp/faq

8. 七牛云图片处理http://blog.qiniu.com/archives/5793

9. https://havecamerawilltravel.com/photographer/webp-website