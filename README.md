# FaceMorphMe
FaceMorph 在Android平台的实现

[![Travis](https://img.shields.io/appveyor/ci/gruntjs/grunt.svg)](https://github.com/huzongyao/FaceMorphMe/releases)
[![Travis](https://img.shields.io/badge/API-16+-brightgreen.svg)](https://github.com/huzongyao/FaceMorphMe)

#### Quick Start

* 查看Face Morphing 效果:

![pic](https://github.com/huzongyao/FaceMorphMe/blob/master/misc/example.gif?raw=true)

* 下载App体验：[下载地址](https://github.com/huzongyao/FaceMorphMe/releases/latest)

#### Details
* FaceMorph是指将一张人脸照片平滑过渡到另一张，
算法主要是参考这篇文章：[Face Morph Using OpenCV](https://www.learnopencv.com/face-morph-using-opencv-cpp-python/)。

* 图像的平滑过渡在电影中有广泛应用，图像过渡的思想也很简单，可以用改变透明度来实现，场景过渡时添加若干半透明中间图片，
透明度参数alpha取0-1，假设需要过渡图片A和图片B，中间图片C=A*(1-alpha) + B*alpha，alpha从0过渡到1，图片便可以从A平滑过渡到B。

* 不过要是对于人像，我们如果能在过度透明度的同时，也慢慢过渡人物五官的位置，这样会让我们得到更加平滑的过渡效果，
如果我找一张我很多年前拍的照片，和我现在的照片做Morph，我甚至可以发现我的相貌哪些地方发生了变化。

##### Compile：
0. 直接编译会报错，需要先下载OpenCV3x的Android开发包,详情见以下说明。
```
make: *** No rule to make target `morpher/src/main/cpp/opencv/jni/OpenCV.mk'.  Stop.
```

1. 关于OpenCV库
* 为了减小生成文件的大小，对OpenCV库采用C++调用，静态链接的方式使用，这样只有被使用到的库函数才会被打包，且不用编译整个OpenCV库。
* 为了减小源码的大小，OpenCV库相关文件使用gitignore排除了，所以编译前需要手动将OpenCV静态库和头文件拷贝到
  morpher/src/main/cpp/opencv目录下。
* OpenCV静态库和头文件的获取有三种方式：</br>
  1.[官方下载](https://opencv.org/releases/),选择3.x版本(目前版本3.4.7需要用旧版NDK才行，不能使用c++_static)</br>
  2.下载我编译的版本：[libopencv-3.4.7.7z](https://github.com/huzongyao/FaceMorphMe/releases/download/v1.0.0/libopencv-3.4.7.7z)(新版本NDK可用)</br>
  3.自己编译一个(先build，再strip，参考misc/script/目录下的脚本)</br>
* 为什么要使用OpenCV3.x而不是4.x：使用到的Stasm库对OpenCV3.x支持较好，不需要改太多代码。

##### Process:
1. 人脸识别，人脸关键点识别
 * OpenCV提供了一个简单的人脸识别功能和一些简单的模型，使用方式简单：
``` c++
CascadeClassifier cascade(classifierPath);
// 如果是彩图可以先转成灰度图
cvtColor(image, gray, CV_RGBA2GRAY);
std::vector<Rect> faces;
cascade.detectMultiScale(gray, faces);
```
 * 人脸关键点识别一般可以采用[Dlib](http://dlib.net/)或者
 [Stasm](http://www.milbo.users.sonic.net/stasm/) + [OpenCV](https://opencv.org/),
 * 使用Dlib的话可以参照[face_landmark_detection](http://dlib.net/face_landmark_detection_ex.cpp.html)
 来获取人脸68个关键点，不依赖于OpenCV，但是需要61M的模型文件。
 * 使用Stasm可以参考示例[minimal](http://www.milbo.users.sonic.net/stasm/minimal.html)
 使用它可以获取77个人脸关键点, 只需要OpenCV的几个xml的模型文件，
 体积比dlib的模型小，比较适合移动端设备使用，所以我选择了这个。接口如下：
```c
int stasm_search_single(     // wrapper for stasm_search_auto and friends
    int*         foundface,  // out: 0=no face, 1=found face
    float*       landmarks,  // out: x0, y0, x1, y1, ..., caller must allocate
    const char*  img,        // in: gray image data, top left corner at 0,0
    int          width,      // in: image width
    int          height,     // in: image height
    const char*  imgpath,    // in: image path, used only for err msgs and debug
    const char*  datadir);   // in: directory of face detector files
```
 * SeetaFace2也是一个可行的方案，可以参考[SeetaFace2](https://github.com/seetafaceengine/SeetaFace2)
  可以实现人脸81个关键点识别，不依赖于OpenCV，效果也不错，不过相比于Stasm，SeetaFace2不会标注人脑门上的关键点。
```c
// 人脸检测
SeetaFaceInfoArray FaceDetector::detect( const SeetaImageData &image);
// 关键点识别
void FaceLandmarker::mark( const SeetaImageData &image, const SeetaRect &face, SeetaPointF *points);
```

2. 点集的三角剖分
 * 所谓Delaunay三角剖分，是一种算法，研究如何把一个离散几何剖分成不均匀的三角形网格，这就是离散点的三角剖分问题，
 散点集的三角剖分，对数值分析以及图形学来说，都是极为重要的一项处理技术。
```c
Rect rect(0, 0, info.width, info.height);
Subdiv2D subDiv(rect);
subDiv.insert(facePts);
std::vector<Point2f> trianglePts;
MorphUtils::getTrianglesPoints(subDiv, trianglePts);
```

3. 三角形变换以及透明度变换
* Alpha变化的同时，加上人脸关键点位置的变换，使得变换更加平滑。
``` c
// 填充凸多边形
void fillConvexPoly(InputOutputArray img, InputArray points,
                 const Scalar& color, int lineType = LINE_8,
                 int shift = 0);
// 仿射变换
void warpAffine( InputArray src, OutputArray dst,
                  InputArray M, Size dsize,
                  int flags = INTER_LINEAR,
                  int borderMode = BORDER_CONSTANT,
                  const Scalar& borderValue = Scalar());
```

4. GIF动图编码
* 使用了开源库[BurstLinker](https://github.com/bilibili/BurstLinker)从Bitmap生成GIF文件
```
burstLinker.init(width, height, filePath);
burstLinker.connect(colorBitmap, BurstLinker.OCTREE_QUANTIZER, BurstLinker.NO_DITHER, 0, 0, delayMs);
burstLinker.release();
```

5. MP4视频编码
* 使用Android统一的MediaCodec硬件编码接口(Android 4.1 (API16) 开始支持)

* 压缩编码格式根据硬件能力依次选择: H.265(HEVC)，H.264(AVC)，MPEG4，H.263

* 编码色彩空间目前支持了YUV420的四种形式：I420， NV12， YV12， NV21。
  很多设备的编码器都支持了YUV420中的一种，YUV420图像所占空间大小为像素的1.5倍，
  如果扩展其他格式的色彩空间可能需要给编码器输入不同大小的数据。
  其中使用了开源库libyuv对RGBA到YUV的转换，libyuv支持cpu硬件加速，比OpenCV或自己写的转换算法运算速度快。

* 测试中发现某些编码器接收的分辨率必须是16的整数倍，所以干脆把设置里的分辨率选项都设置为16的整倍数。

* 使用Android统一的MediaMuxer混频封装接口(Android 4.3 (API18) 开始支持)，输出为MP4格式文件。

#### Remain Problems
1. 使用uCrop裁剪图片，不能固定输出图片的分辨率，只能指定最大分辨率，因此会有图片过小的情况。
2. 视频生成依赖于系统的MediaCodec接口，不同的设备表现会不一致，有可能崩溃。
3. 人脸关键点识别基于开源的stasm，效果多半没有商用产品如Face++那么好。
4. stasm人脸关键点坐标可能位于图片之外，这会导致三角剖分崩溃或出现异常，所以人脸要尽量位于图片中。

#### Reference
1. https://www.learnopencv.com/face-morph-using-opencv-cpp-python/
2. https://github.com/Yalantis/uCrop
3. https://github.com/bilibili/BurstLinker
4. https://github.com/brianwernick/ExoMedia
5. https://github.com/seetafaceengine/SeetaFace2

### About Me
 * GitHub: [https://huzongyao.github.io/](https://huzongyao.github.io/)
 * ITEye博客：[https://hzy3774.iteye.com/](https://hzy3774.iteye.com/)
 * 新浪微博: [https://weibo.com/hzy3774](https://weibo.com/hzy3774)

### Contact To Me
 * QQ: [377406997](https://wpa.qq.com/msgrd?v=3&uin=377406997&site=qq&menu=yes)
 * Gmail: [hzy3774@gmail.com](mailto:hzy3774@gmail.com)
 * Foxmail: [hzy3774@qq.com](mailto:hzy3774@qq.com)
 * WeChat: hzy3774

 ![image](https://raw.githubusercontent.com/hzy3774/AndroidP7zip/master/misc/wechat.png)

### Others
 * 想捐助我喝杯热水(¥0.01起捐)</br>
 ![donate](https://github.com/huzongyao/JChineseChess/blob/master/misc/donate.png?raw=true)
