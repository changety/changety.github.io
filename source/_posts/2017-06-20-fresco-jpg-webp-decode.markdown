---
layout: post
title: "Fresco图片解码部分源码分析及webp vs jpeg指标对比"
date: 2017-06-20 16:04:07 +0800
comments: true
categories: Android Fresco decode Webp Jpeg
---

#前言
*之前的文章写过[webp图片的调研](http://changety.github.io/blog/2016/01/31/webp-research/)，这篇分析一下fresco的decoder部分的源码，同时从响应、下载、解码、大小四个指标上对比同一张图片的webp 与jpg格式。这里响应时间应该与图片格式本身没有关系，但这里为了对服务器接口做一个测试也加入了对比；下载时间应该与图片size成正相关，这里也加入对比，看看结果是否符合预期。根据google官网介绍，目前WebP与JPG相比较，编解码速度上，毫秒级别上：编码速度webp比jpg慢10倍，解码速度慢1.5倍。在我们的使用场景下，编码速度的影响可以被忽略，因为服务器会在用户第一次请求时，编码生成jpg图片对应的webp图片，之后都会被缓存下来，可以认为几乎所有用户的请求都能命中缓存。解码方面，则是每个用户拿到webp图片都必经的开销，因此解码速度是本次测试对比的关键指标。*

#Fresco WebP支持
我们的客户端使用的是[fresco](https://github.com/facebook/fresco)图片库，根据其[官方文档说明](http://frescolib.org/docs/webp-support.html#main_wrap)：

>Android added webp support in version 4.0 and improved it in 4.2.1:
4.0+ (Ice Cream Sandwich): basic webp support
4.2.1+ (Jelly Beam MR1): support for transparency and losless wepb
Fresco handles webp images by default if the OS supports it. So you can use webp with 4.0+ and trasparency and losless webps from 4.2.1.
Fresco also supports webp for older OS versions. The only thing you need to do is add thewebpsupportlibrary to your dependencies. So if you want to use webps on Gingerbread just add the following line to your gradle build file:
*compile 'com.facebook.fresco:webpsupport:1.3.0'*

因此我们需要引入webpsupprot库，这样子fresco会处理对webp的支持。下面也会从源码上分析，fresco是如何解码webp的。

#Fresco Producer源码分析
##Producer继承结构
首先我们看一下Frecso中Producer的继承结构图：



![fresco producer继承关系](http://upload-images.jianshu.io/upload_images/76332-66b1c9b26eb0033d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##Producer流水线

`ProducerSequenceFactory`是专门将生成各类链接起来的Producer，根据其中的逻辑，这里将可能涉及层次最深的Uri——网络Uri的Producer链在此列出，它会到每个缓存中查找数据，最后如果都没有命中，则会去网络上下载。

|顺序|Producer|是否必须|功能|
|:--:|:--:|:--:|:--|
|1|PostprocessedBitmapMemoryCacheProducer|否|在Bitmap缓存中查找被PostProcess过的数据|
|2|PostprocessorProducer|否|对下层Producer传上来的数据进行PostProcess|
|3|BitmapMemoryCacheGetProducer|是|使Producer序列只读|
|4|ThreadHandoffProducer|是|使下层Producer工作在后台进程中执行|
|5|BitmapMemoryCacheKeyMultiplexProducer|是|使多个相同已解码内存缓存键的ImageRequest都从相同Producer中获取数据|
|6|BitmapMemoryCacheProducer|是|从已解码的内存缓存中获取数据|
|7|**DecodeProducer**|是|将下层Producer产生的数据解码|
|8|ResizeAndRotateProducer|否|将下层Producer产生的数据变换|
|9|EncodedCacheKeyMultiplexProducer|是|使多个相同未解码内存缓存键的ImageRequest都从相同Producer中获取数据|
|10|EncodedMemoryCacheProducer|是|从未解码的内存缓存中获取数据|
|11|DiskCacheProducer|是|从文件缓存中获取数据|
|12|WebpTranscodeProducer|否|Transcodes WebP to JPEG / PNG|
|13|**NetworkFetchProducer**|是|从网络上获取数据|
****
<br />
为了获得每一张网络图片的大小、响应时间、下载时间、decode时间，我们需要探索Fresco的源码，挂上钩子去得到这些指标；这里我们关心`DecoderProducer`、`NetworkFetchProducer`，顾名思义，这两个Producer分别用于解码和网络加载相关。

###DecodeProducer解码过程
DecodeProducer负责将未解码的数据生产出解码的数据。先看produceResults方法。
```  
  @Override
  public void produceResults(final Consumer<CloseableReference<CloseableImage>> consumer, final ProducerContext producerContext) {
    final ImageRequest imageRequest = producerContext.getImageRequest();
    ProgressiveDecoder progressiveDecoder;
    if (!UriUtil.isNetworkUri(imageRequest.getSourceUri())) {
      progressiveDecoder = new LocalImagesProgressiveDecoder(consumer, producerContext, mDecodeCancellationEnabled);
    } else {
      ProgressiveJpegParser jpegParser = new ProgressiveJpegParser(mByteArrayPool);
      progressiveDecoder = new NetworkImagesProgressiveDecoder(consumer, producerContext, jpegParser, mProgressiveJpegConfig, mDecodeCancellationEnabled);
    }
    mInputProducer.produceResults(progressiveDecoder, producerContext);
  }```

通过判断uri的类型 选择不同的渐近式解释器，local和network都继承自ProgressiveDecoder

在`ProgressiveDecoder`的构造方法中，doDecode(encodedImage, isLast) 进行解析。而真正解析的则是`ImageDecode`r#decodeImage方法，这个方法将encodedImage解析成`CloseableImage`：
```Java
    /** Performs the decode synchronously. */
    private void doDecode(EncodedImage encodedImage, @Status int status) {
      if (isFinished() || !EncodedImage.isValid(encodedImage)) {
        return;
      }
      final String imageFormatStr;
      ImageFormat imageFormat = encodedImage.getImageFormat();
      if (imageFormat != null) {
        imageFormatStr = imageFormat.getName();
      } else {
        imageFormatStr = "unknown";
      }
      final String encodedImageSize;
      final String sampleSize;
      final boolean isLast = isLast(status);
      final boolean isLastAndComplete = isLast && !statusHasFlag(status, IS_PARTIAL_RESULT);
      final boolean isPlaceholder = statusHasFlag(status, IS_PLACEHOLDER);
      if (encodedImage != null) {
        encodedImageSize = encodedImage.getWidth() + "x" + encodedImage.getHeight();
        sampleSize = String.valueOf(encodedImage.getSampleSize());
      } else {
        // We should never be here
        encodedImageSize = "unknown";
        sampleSize = "unknown";
      }
      final String requestedSizeStr;
      final ResizeOptions resizeOptions = mProducerContext.getImageRequest().getResizeOptions();
      if (resizeOptions != null) {
        requestedSizeStr = resizeOptions.width + "x" + resizeOptions.height;
      } else {
        requestedSizeStr = "unknown";
      }
      try {
        long queueTime = mJobScheduler.getQueuedTime();
        long decodeDuration = -1;
        String imageUrl = encodedImage.getEncodedCacheKey().getUriString();
        int length = isLastAndComplete || isPlaceholder ? encodedImage.getSize() : getIntermediateImageEndOffset(encodedImage);
        QualityInfo quality = isLastAndComplete || isPlaceholder ? ImmutableQualityInfo.FULL_QUALITY : getQualityInfo();

        mProducerListener.onProducerStart(mProducerContext.getId(), PRODUCER_NAME);
        CloseableImage image = null;
        try {
          long nowTime = System.currentTimeMillis();
          image = mImageDecoder.decode(encodedImage, length, quality, mImageDecodeOptions);
          decodeDuration = System.currentTimeMillis() - nowTime;
        } catch (Exception e) {
          Map<String, String> extraMap = getExtraMap(image, imageUrl, queueTime, decodeDuration, quality, isLast, imageFormatStr, encodedImageSize, requestedSizeStr, sampleSize);
          mProducerListener.onProducerFinishWithFailure(mProducerContext.getId(), PRODUCER_NAME, e, extraMap);
          handleError(e);
          return;
        }
        Map<String, String> extraMap = getExtraMap(image, imageUrl, queueTime, decodeDuration, quality, isLast, imageFormatStr, encodedImageSize, requestedSizeStr, sampleSize);
        mProducerListener.onProducerFinishWithSuccess(mProducerContext.getId(), PRODUCER_NAME, extraMap);
        handleResult(image, status);
      } finally {
        EncodedImage.closeSafely(encodedImage);
      }
    }
```
因此我们在#doDecoder方法在decode前后插入解码时长计算。
```
  long nowTime = System.currentTimeMillis();
  image = mImageDecoder.decode(encodedImage, length, quality, mImageDecodeOptions);
  decodeDuration = System.currentTimeMillis() - nowTime;
```

####ImageDecoder
`DecoderProducer` 中是依赖`ImageDecoder `类，用来将未解码的`EncodeImage`解码成对应的`CloseableImage`。`ImageDecoder`中先判断未解码的图片类型：
```  
  private final ImageDecoder mDefaultDecoder = new ImageDecoder() {
    @Override
    public CloseableImage decode(EncodedImage encodedImage, int length, QualityInfo qualityInfo, ImageDecodeOptions options) {
      ImageFormat imageFormat = encodedImage.getImageFormat();
      if (imageFormat == DefaultImageFormats.JPEG) {
        return decodeJpeg(encodedImage, length, qualityInfo, options);
      } else if (imageFormat == DefaultImageFormats.GIF) {
        return decodeGif(encodedImage, length, qualityInfo, options);
      } else if (imageFormat == DefaultImageFormats.WEBP_ANIMATED) {
        return decodeAnimatedWebp(encodedImage, length, qualityInfo, options);
      } else if (imageFormat == ImageFormat.UNKNOWN) {
        throw new IllegalArgumentException("unknown image format");
      }
      return decodeStaticImage(encodedImage, options);
    }
  };
```  

####ImageFormatChecker
这个类是根据输入流来确定图片的类型。基本原理是根据头标识去确定类型。根据代码能看出，这里分为几种。
```
  public static final ImageFormat JPEG = new ImageFormat("JPEG", "jpeg");
  public static final ImageFormat PNG = new ImageFormat("PNG", "png");
  public static final ImageFormat GIF = new ImageFormat("GIF", "gif");
  public static final ImageFormat BMP = new ImageFormat("BMP", "bmp");
  public static final ImageFormat WEBP_SIMPLE = new ImageFormat("WEBP_SIMPLE", "webp");
  public static final ImageFormat WEBP_LOSSLESS = new ImageFormat("WEBP_LOSSLESS", "webp");
  public static final ImageFormat WEBP_EXTENDED = new ImageFormat("WEBP_EXTENDED", "webp");
  public static final ImageFormat WEBP_EXTENDED_WITH_ALPHA = new ImageFormat("WEBP_EXTENDED_WITH_ALPHA", "webp");
  public static final ImageFormat WEBP_ANIMATED = new ImageFormat("WEBP_ANIMATED", "webp");
```
本篇我们关心以下几种：
- JPEG
- WEBP_SIMPLE
- GIF
- WEBP_ANIMATED

从是否静态图上来看，为两种：
- 可动 ，用AnimatedImageFactory进行解析
- 不可动，用PlatformDecoder进行解析

#### AnimatedImageFactory
AnimatedImageFactory是一个接口，他的实现类是AnimatedImageFactoryImpl。
在这个类的静态方法块种，通过如下代码 来构造其他依赖包中的对象，这个小技巧我们可以get一下。
```
  private static AnimatedImageDecoder loadIfPresent(final String className) {
    try {
      Class<?> clazz = Class.forName(className);
      return (AnimatedImageDecoder) clazz.newInstance();
    } catch (Throwable e) {
      return null;
    }
  }

  static {
    sGifAnimatedImageDecoder = loadIfPresent("com.facebook.animated.gif.GifImage");
    sWebpAnimatedImageDecoder = loadIfPresent("com.facebook.animated.webp.WebPImage");
  }
```

AnimatedImageDecoder又分别有两个实现：
- `WebpImage`
- `GifImage`

![WebpImage与GifImage](http://upload-images.jianshu.io/upload_images/76332-9ce98211c5be0f50.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解析分为两个步骤：
1. 通过`AnimatedImageDecode`r解析出AnimatedImage
2. 利用getCloseableImage从`AnimatedImage`中构造出`CloseableAnimatedImage`。这是`CloseableImage`的之类。
getCloseableImage的逻辑如下：
1. 用decodeAllFrames解析出所有帧
2. 用createPreviewBitmap构造预览的bitmap
3. 构造`AnimatedImageResult`对象
4. 用`AnimatedImageResult`构造`CloseableAnimatedImage`对象。

####PlatformDecoder
`PlatformDecoder`是一个接口，代表不同平台。我们看他的实现类有哪些:


![PlatformDecoder具体实现](http://upload-images.jianshu.io/upload_images/76332-37a7a0bdcb84bc8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
public interface PlatformDecoder {
  /**
   * Creates a bitmap from encoded bytes. Supports JPEG but callers should use {@link
   * #decodeJPEGFromEncodedImage} for partial JPEGs.
   *
   * @param encodedImage the reference to the encoded image with the reference to the encoded bytes
   * @param bitmapConfig the {@link android.graphics.Bitmap.Config} used to create the decoded
   * Bitmap
   * @return the bitmap
   * @throws TooManyBitmapsException if the pool is full
   * @throws java.lang.OutOfMemoryError if the Bitmap cannot be allocated
   */
  CloseableReference<Bitmap> decodeFromEncodedImage(final EncodedImage encodedImage, Bitmap.Config bitmapConfig);

  /**
   * Creates a bitmap from encoded JPEG bytes. Supports a partial JPEG image.
   *
   * @param encodedImage the reference to the encoded image with the reference to the encoded bytes
   * @param bitmapConfig the {@link android.graphics.Bitmap.Config} used to create the decoded
   * Bitmap
   * @param length the number of encoded bytes in the buffer
   * @return the bitmap
   * @throws TooManyBitmapsException if the pool is full
   * @throws java.lang.OutOfMemoryError if the Bitmap cannot be allocated
   */
  CloseableReference<Bitmap> decodeJPEGFromEncodedImage(EncodedImage encodedImage, Bitmap.Config bitmapConfig, int length);
}


getBitmapFactoryOptions 获取BitmapFactory.Options
decodeByteArrayAsPurgeable 获取bitmap
pinBitmap 真正的decode
```

#NetworkFetchProducer
`NetworkFetchProducer`负责从网络层获取图片流，持有`NetworkFetcher`的实现类；测试代码中，我们添加了OkHttp3的`OkHttpNetworkFetcher`作为fetcher；我们关心`NetworkFetchProducer`中的这个方法：

```
  private void handleFinalResult(PooledByteBufferOutputStream pooledOutputStream, FetchState fetchState) {
    Map<String, String> extraMap = getExtraMap(fetchState, pooledOutputStream.size());
    ProducerListener listener = fetchState.getListener();
    listener.onProducerFinishWithSuccess(fetchState.getId(), PRODUCER_NAME, extraMap);
    listener.onUltimateProducerReached(fetchState.getId(), PRODUCER_NAME, true);
    notifyConsumer(pooledOutputStream, Consumer.IS_LAST | fetchState.getOnNewResultStatusFlags(), fetchState.getResponseBytesRange(), fetchState.getConsumer());
  }
```
该方法中`FetchState`记录了一张图片从服务端响应到IO读取的耗时记录。同样的，也是通过`ProducerListener`的#onProducerFinishWithSuccess方法回调出去。

#计算decode & fecth的时间

每个`Producer`的接口实现，都会持有`ProducerContext`，其中的`ProducerListener`会回调`Producer`各个阶段的事件。我们关心这个方法：

>/**
   >* Called when a producer successfully finishes processing current unit of work.
   >* @param extraMap Additional parameters about the producer. This map is immutable and will
   >* throw an exception if attempts are made to modify it.
   >*/
  >void onProducerFinishWithSuccess(String requestId, String producerName, @Nullable Map<String, >String> extraMap);

该方法会在`Producer`结束时回调出来，我们利用Fresco包里的`RequestLoggingListener`，便可监听到`DecoderProducer`和`NetworkFetchProducer`的回调。

```
    @Override
    public void onCreate() {
        super.onCreate();
        FLog.setMinimumLoggingLevel(FLog.VERBOSE);
        Set<RequestListener> listeners = new HashSet<>(1);
        listeners.add(new RequestLoggingListener());
        ImagePipelineConfig config = ImagePipelineConfig.newBuilder(this)
            .setRequestListeners(listeners)
            .setNetworkFetcher(new OkHttpNetworkFetcher(OKHttpFactory.getInstance().getOkHttpClient()))
            .build();
        DraweeConfig draweeConfig = DraweeConfig.newBuilder().setDrawDebugOverlay(DebugOverlayHelper.isDebugOverlayEnabled(this)).build();
        Fresco.initialize(this, config, draweeConfig);
        Fresco.getImagePipeline().clearDiskCaches();
    }
```

我们通过在Fresco初始化Builder中加入`RequestLoggingListener`，并改造`RequestLoggingListener`的onProducerFinishWithSuccess方法:
```
  @Override
  public synchronized void onProducerFinishWithSuccess(String requestId, String producerName, @Nullable Map<String, String> extraMap) {
    if (FLog.isLoggable(FLog.VERBOSE)) {
      Pair<String, String> mapKey = Pair.create(requestId, producerName);
      Long startTime = mProducerStartTimeMap.remove(mapKey);
      long currentTime = getTime();
      long producerDuration = -1;
      FLog.v(TAG, "time %d: onProducerFinishWithSuccess: " + "{requestId: %s, producer: %s, elapsedTime: %d ms, extraMap: %s}", currentTime, requestId, producerName, producerDuration = getElapsedTime(startTime, currentTime), extraMap);
      if (sOnProducer != null) {
        sOnProducer.onProducer(producerDuration, extraMap, producerName);
      }
    }
  }
```
通过将Producer的信息回调给外面，至此我们就拿到了每一个Producer的回调信息，通过producerName的过滤就可以拿到关心的信息，这里我们关心`DecoderProducer`和`NetworkFetchProducer`的信息。

###decode时间计算
在`DecoderProducer`的doDecode方法中插入:
```
 long nowTime = System.currentTimeMillis();
 image = mImageDecoder.decode(encodedImage, length, quality, mImageDecodeOptions);
 decodeDuration = System.currentTimeMillis() - nowTime;
```
decodeDuration放入onProducerFinishWithSuccess的extraMap当中

###network fecther时间计算
`OkHttpNetworkFetcher`中定义着几个常量值：
```
  public static final String QUEUE_TIME = "resp_time"; //修改为响应时间
  public static final String FETCH_TIME = "fetch_time";
  public static final String TOTAL_TIME = "total_time";
  public static final String IMAGE_SIZE = "image_size";
  public static final String IMAGE_URL = "image_url"; //新增
```
- `QUEUE_TIME`为请求丢入请求线程池到最后请求成功响应的时间
- `FETCH_TIME`为从response读完IO流的时间
- `IMAGE_SIZE`为response header中的content-length，即图片大小

这些数据都最终会被丢入extraMap，回调给外面。

#数据对比

##jpg&webp指标对比:



![jpg webp指标对比](http://upload-images.jianshu.io/upload_images/76332-f960ec0cb0bed228.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

|格式|图片数|大小|响应时间|下载时间|解码时间|总用时|
|:--:|:--:|:--:|:--|:--:|:--:|:--:|
|jpg|100|33497748B / 31.9MB|3384ms|5582ms|7225ms|16191ms
|webp|100|11127628B / 10.6MB |3388ms|2552ms|9806ms|15746ms

以上数据经过几轮测试，都接近这个数据对比。 图片源来自项目线上的图片，图片接口来自公司CND接口，使用相同quality参数，带同一张图片的不同格式参数。解码总时间大概是JPG:WEBP = 1 : 1.3左右，接近官方的1.5倍性能差距。总大小上，webp几乎只有jpg的1/3，远超超官方的30%，这个估计是大多数jpg没有经过压缩就直接上传了。下载时间基本上与size成正比。

http://p1.music.126.net/6OARlbfxOysQJU5iZ8WKSA==/18769762999688243.jpg?imageView=1&type=webp&quality=100

[http://p1.music.126.net/6OARlbfxOysQJU5iZ8WKSA==/18769762999688243.jpg?imageView=1&quality=100](http://p1.music.126.net/6OARlbfxOysQJU5iZ8WKSA==/18769762999688243.jpg?imageView=1&quality=100)


##gif&anim-webp指标对比:

![gif&anim-webp指标对比](http://upload-images.jianshu.io/upload_images/76332-114e6bf70693178d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

|格式|图片数|大小|响应时间|下载时间|解码时间|总用时|
|:--:|:--:|:--:|:--|:--:|:--:|:--:|
|gif|85|66343142B / 63.26MB|2597ms|6052ms|272ms|8921ms
|anim-webp|85|20342068B / 19.39MB|2687ms|3809ms|240ms|6736ms

同样地, 分别取了同一张GIF图片的，原始版本与WEBP版本来对比。

[http://p1.music.126.net/rhGo28bJP19-T0xmtpg6jw==/19244752021149272.jpg](http://p1.music.126.net/rhGo28bJP19-T0xmtpg6jw==/19244752021149272.jpg)
[http://p1.music.126.net/rhGo28bJP19-T0xmtpg6jw==/19244752021149272.jpg?imageView=1&type=webp&tostatic=0](http://p1.music.126.net/rhGo28bJP19-T0xmtpg6jw==/19244752021149272.jpg?imageView=1&type=webp&tostatic=0)

size压缩对比也接近1：3；另外这里的解码时间是不准确的，因为webp与gif在fresco中都是`AnimatedImage`，他们的decode调的是nativeCreateFromNativeMemory方法，这个方法返回是对应的`WebPImage` 与`GifImage`对象，表中的解码时间也是构建这个对象的耗时；动图渲染时，主要调用的是`AnimatedDrawableBackendImpl`中renderFrame方法。但我们可以粗略认为，每一帧的渲染耗时对比，接近jpg与webp的耗时；因为gif与anim-webp分别是由一帧一帧的jpg与webp组成。

#总结

webp与jpg相比，包括anim-webp与gif， 在相同的图片质量，图片大小上，webp有着巨大的优势，解码速度毫秒级的差距也完全在接收范围内， 而图片大小最终转化为带宽、存储空间、加载速度上的优势。因此在有条件的情况，app中完全可以用webp来替代jpg格式以提升加载速度、降低存储空间、节省带宽费用。另外在android上，使用fresco作为图片库，可以几乎无成本的接入webp。

参考
1.http://changety.github.io/blog/2016/01/31/webp-research/
2.https://developers.google.com/speed/webp/faq
3.https://guolei1130.github.io/2016/12/13/fresco%E5%9B%BE%E7%89%87decode%E7%9A%84%E5%A4%A7%E4%BD%93%E6%B5%81%E7%A8%8B/