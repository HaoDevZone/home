### 如何理解一张图片？
图片是像素组成的，像素简称px。通常图片像素是按RGB顺序排列，1个像素对应R（红色）、G（绿色）、B（蓝色）这3个通道，也有ARGB
，增加了A（透明度）通道。

有RGBA组成的一个像素，占用多大空间呢？
一般来说每个通道占用1个字节（byte），也就是8bit，一个像素也就是占用4个字节=32bit，对于一张图片， 未压缩大小 (字节)=宽度×高度×4 。
由此可见，占用的空间很大，我们通常会对图片进行压缩。

### CameraX 和 YUV_420
- CameraX：
    - CameraX 支持Android 5.0 (API level 21)以及更高版本, 覆盖了98%的Android设备.
    - Android 的现代化相机 API，提供简化的图像捕获接口。
    - 默认输出格式为 YUV_420_888（灵活的 YUV 格式，支持多种子采样）。
- YUV_420 格式：
  - 包含亮度（Y）通道和两个色度（U、V）通道，色度通常以 4:2:0 采样（节省空间）。
  - 数据结构：三个平面（Y、U、V），或部分交错。
  - 示例：1080p 图像 ≈ 3MB（Y: 1920x1080，U/V: 960x540）。
- RGB 格式：
  - 每个像素存储红、绿、蓝值（通常 32 位 RGBA）。
  - 示例：1080p 图像 ≈ 8MB（1920x1080x4）。
  - 用途：图像处理、显示、JNI 调用（如 OpenGL、机器学习模型）。


### NV21
这是相机预览图像的默认格式，除非使用Camera.Parameters.setPreviewFormat(int)另行设置。 不过在
android.hardware.camera2 API中，建议使用 YUV_420_888 格式作为 YUV 输出。

### 图片处理
通常我们会配合opencv对图片进行各种处理，因此涉及到NDK的开发，下面就列举在开发中需要掌握的知识点
1. CameraX
   - Preview: View an image on the display.翻译过来就叫做预览画面。
   - Image analysis: Access a buffer seamlessly for use in your algorithms, such as to pass to ML Kit.这个非常重要，我们可以
   拿到buffer，用它来做进一步处理，例如他说的传递给 ML Kit（机器学习工具包），开发者可以利用其强大的机器学习模型来实现各种智能功能，比如图像识别、文本识别和更多。
      - 阻塞 ：适合Image analysis性能高，延迟低的情况下，因为在该模式下图像会存在内部队列当中，只有队列满的时候才开始丢帧。
      - 非阻塞：默认选项，始终会将最新的图像缓存到图像缓冲区。
      - 示例将 CameraX ImageAnalysis 和 Preview 用例绑定到了 lifeCycle 所有者：
        ``` 
        val imageAnalysis = ImageAnalysis.Builder()
        // enable the following line if RGBA output is needed.
        // .setOutputImageFormat(ImageAnalysis.OUTPUT_IMAGE_FORMAT_RGBA_8888)
        .setTargetResolution(Size(1280, 720))
        .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
        .build()
        imageAnalysis.setAnalyzer(executor, ImageAnalysis.Analyzer { imageProxy ->
           val rotationDegrees = imageProxy.imageInfo.rotationDegrees
          // insert your code here.
          ...
          // after done, release the ImageProxy object
          imageProxy.close()
        })
        
        cameraProvider.bindToLifecycle(this as LifecycleOwner, cameraSelector, imageAnalysis, preview)
        ```
        
2. ByteBuffer
   // YUV_420_888格式，打印
   ``` 
    image format: 35
    pixelStride 1
    rowStride 1920
    width 1920
    height 1080
    buffer size 2073600
    Finished reading data from plane 0
    pixelStride 1
    rowStride 960
    width 1920
    height 1080
    buffer size 518400
    Finished reading data from plane 1
    pixelStride 1
    rowStride 960
    width 1920
    height 1080
    buffer size 518400
    Finished reading data from plane 2
   ```
   分析：   
   根据image的分辨率是 1920 x 1080 ，像素点个数是2073600 。下面分别对plane[0]、plane[1]、plane[2]作分析。
    - plane[0]表示Y，rowStride是1920 ，其pixelStride是1 ，说明Y存储时中间无间隔，每行1920个像素全是Y值，buffer size 是 plane[0]的1/4 ，buffer size / rowStride= 1080可知Y有1080行。
    - plane[1]表示U，rowStride是960 ，其pixelStride也是1，说明连续的U之间没有间隔，每行只存储了960个数据，buffer size 是 plane[0]的1/4 ，buffer size / rowStride = 540 可知U有540行，对于U来说横纵都是1/2采样。
    - pane[2]和plane[1]相同。、
 应用可以将输出图像像素配置为采用 YUV（默认）或 RGBA 颜色空间。设置 RGBA 输出格式时，CameraX 会在内部将图像从 YUV 颜色空间转换为 RGBA 颜色空间，并将图像位打包到 ImageProxy 第一个平面（其他两个平面未使用）的 ByteBuffer 中，序列如下：
   ``` 
   ImageProxy.getPlanes()[0].buffer[0]: alpha
   ImageProxy.getPlanes()[0].buffer[1]: red
   ImageProxy.getPlanes()[0].buffer[2]: green
   ImageProxy.getPlanes()[0].buffer[3]: blue
   ... 
   ```
   通常我们会采用YUV（默认）进行处理
   ```kotlin
    val yBuffer = image.planes[0].buffer
    val uBuffer = image.planes[1].buffer
    val vBuffer = image.planes[2].buffer
    val yData = ByteArray(yBuffer.remaining())
    val uData = ByteArray(uBuffer.remaining())
    val vData = ByteArray(vBuffer.remaining())
    yBuffer.get(yData)
    uBuffer.get(uData)
    vBuffer.get(vData)
    
    // JNI 调用
    processYuvImage(
        yData, uData, vData,
        image.width, image.height,
        image.planes[0].rowStride,
        image.planes[1].rowStride,
        image.planes[2].rowStride,
        isFrontCamera
    )
   ```
   //这里可以进行优化，直接传递ByteBuffer，少了copy数组的操作，节省 ~0.5ms在1080P。
   ```kotlin
    val yBuffer = image.planes[0].buffer
    val uBuffer = image.planes[1].buffer
    val vBuffer = image.planes[2].buffer 
    
    // JNI 调用
    processYuvImage(
        yBuffer, uBuffer, vBuffer,
        image.width, image.height,
        image.planes[0].rowStride,
        image.planes[1].rowStride,
        image.planes[2].rowStride,
        lensFacing
    )
   ```
3. JNI调用   
  java
    ```java
    public native void processYuvImage(ByteBuffer yBytes, ByteBuffer uBytes, ByteBuffer vBytes, int width, int height, int yStride, int uStride, int vStride, int lensFacing);
    ```
   C++
   
   ```c
   /*
   witch 宽度 代表有多少列cols    
   height 高度 代表有多少行rows
   Mat(int rows, int cols, int type, void* data, size_t step=AUTO_STEP);
   
   YUV_420_888 格式：一个像素一个Y，四个Y共用一个U、V       
   Y Y Y Y    
   Y Y Y Y   
   U U   
   V V 
   因此：*/
   
   width = 4，height = 4
   
   
   //获取 YUV 数据
   uint8_t *y = (uint8_t *) env->GetDirectBufferAddress(y_bytes);
   uint8_t *u = (uint8_t *) env->GetDirectBufferAddress(u_bytes);
   uint8_t *v = (uint8_t *) env->GetDirectBufferAddress(v_bytes);
   
   //创建 Mat 对象   
   cv::Mat yMat(height, y_stride, CV_8UC1, y);
   cv::Mat uMat(height/2, u_stride, CV_8UC1, u);
   cv::Mat vMat(height/2, v_stride, CV_8UC1, v);
   
   //裁剪到实际尺寸（去除 stride 填充）
   
   ```
4. OpenCV:   
     
 
    ```c
    // 裁剪到实际尺寸（去除 stride 填充）
    cv::Mat yMatCropped = yMat(cv::Rect(0, 0, width, height));
    ```
   
暂时不写了，发现同样是 YUV_420_888 格式，但内部还区分I420和NV21，这样处理起来比较麻烦，
参考：https://www.codeleading.com/article/1119284993/
https://cloud.tencent.com/developer/article/1747685
https://markrepo.github.io/avcodec/2018/06/28/YUV/
https://www.jianshu.com/p/944ede616261

I  pixelStride  1
I  rowStride   640
I  width  640
I  height  480
I  Finished reading data from plane  0
I  i=1,size = 153599
I  pixelStride  2
I  rowStride   640
I  width  640
I  height  480
I  Finished reading data from plane  1
I  i=2,size = 153599
I  pixelStride  2
I  rowStride   640
I  width  640
I  height  480
I  Finished reading data from plane  2
I  i=0,size = 307200
I  pixelStride  1
I  rowStride   640
I  width  640
I  height  480
I  Finished reading data from plane  0
I  i=1,size = 153599
I  pixelStride  2
I  rowStride   640
I  width  640
I  height  480
I  Finished reading data from plane  1
I  i=2,size = 153599
I  pixelStride  2
I  rowStride   640
I  width  640
I  height  480



