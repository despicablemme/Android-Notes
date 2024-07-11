# camera demo

## 开发流程
1. 新建Manifest。新建activity，对应页面布局layout；
2. 如果需要，在`build.gradle(app)`中添加所需依赖（如camerax），配置相关项，如打开viewbinding;
3. manifest中注册所需权限，activity中代码申请权限；
4. 布局中添加所需组件，代码中对应activity状态内实现逻辑；

## 绑定视图
1. 在build.gradle的android空间中中添加配置项：
```
buildFeatures {
        viewBinding = true
    }
```
2. activity代码中导入对应的布局类,类的名称方式由activity和layout名称决;使用inflate方法
得到一个bind实例。该实例就包含了layout中的所有元素对象。
```
import com.example.myapplication.databinding.ActivityMainBinding;
// 实例化binding对象
ActivityMainBinding binding;
binding = ActivityMainBinding.inflate(getLayoutInflater());
// 得到布局
View view = binding.getRoot();
// 得到对应元素
Button button = binding.closeButton;
```

## Camerax
1. Camerax是在camera2实现的一个相机类，可以方便调用相机功能。
2. 在`build.gradle(app)`中添加camerax依赖项：
```
val cameraxVersion = "1.3.4"
implementation("androidx.camera:camera-core:${cameraxVersion}")
implementation("androidx.camera:camera-camera2:${cameraxVersion}")
implementation("androidx.camera:camera-lifecycle:${cameraxVersion}")
implementation("androidx.camera:camera-video:${cameraxVersion}")
implementation("androidx.camera:camera-view:${cameraxVersion}")
implementation("androidx.camera:camera-extensions:${cameraxVersion}")
```
3. 重新sync项目，gradle会为项目自动下载对应的依赖到项目中；
4. 重要的几个类
```
import androidx.camera.lifecycle.ProcessCameraProvider;   // 用于获取一个cameraProvider,绑定相机生命周期
import androidx.camera.core.Camera;   // camera类，cameraProvider绑定cameraSelector和Preview的生命周期后返回一个camera实例
import androidx.camera.core.CameraSelector;  // 用于选择前后置相机
import androidx.camera.core.CameraControl;  // 用于控制camera，变焦对焦等
import androidx.camera.core.CameraInfo;   // 用于获取camera信息，焦距，倍数等
import androidx.camera.core.ZoomState;   // zoom状态类
import androidx.camera.view.PreviewView;  // layout组件中preview类，自适应预览相机画面
import androidx.camera.core.Preview;   // 预览类，用来连接一个PreviewView组件
```
5. 获取相机预览方法
```
// 获取cameraProviderFuture实例，由模板ListenableFuture实例化来
ListenableFuture<ProcessCameraProvider> cameraProviderFuture = ProcessCameraProvider.getInstance(this);
// 从future get到provider实例
ProcessCameraProvider cameraProvider = cameraProviderFuture.get();
// 得到一个cameraSelector,传入前后设的宏定义
CameraSelector cameraSelector = new CameraSelector.Builder()
                .requireLensFacing(CameraSelector.LENS_FACING_FRONT).build();
// preview连接
Preview preview = new Preview.builder().build();
preview.setSurfaceProvider(binding.myPreviewView.getSurfaceProvider());
// 绑定生命周期，得到当前调用camera实例
Camera camera = cameraProvider.bindToLifeCycle(this, cameraSelector, preview);
```
6. 相机控制与信息获取
```
CameraControl cameraControl = camera.getCameraControl();
// cameraInfo获取方法相同
```

## Matrix
矩阵类，可用于图像处理，翻转、缩放等等。
```
Matrix matrix = new Matrix();
matrix.setRotate();
matrix.setScale(ratio_width, ratio_height);
// use case1
Bitmap bitmapn = Bitmap.createBitmap(bitmap, xbias, ybias, outputSize.getWidth(), outputSize.getHeight(), matrix, false);
// use case2
canvas.drawBitmap(bitmap, matrix, new Paint());
```

## Future，ListenableFuture，获取cameraProvider的过程

## 图像、摄像头、屏幕旋转
[屏幕旋转时的activity总结](https://chanthuang.github.io/2016/05/19/Android-%E6%A8%AA%E7%AB%96%E5%B1%8F%E5%A4%84%E7%90%86%E7%9A%84%E7%9F%A5%E8%AF%86%E5%B0%8F%E7%BB%93/)

[Google：图像旋转角度、相机物理角度](https://developer.android.com/media/camera/camerax/orientation-rotation?hl=zh-cn)

三个要素：

  默认相机正面竖置为参考点。

  屏幕旋转角度（逆时针旋转角度）；相机物理角度（需要多少顺时针能摆正图片）；图像旋转角度（待顺时针旋转角度）。

## 问题记录
1. 构建cameraSetector的时候为什么不能直接传递0或1，要传预定义进去，值是一样的？
2. 为什么在onCreate中获取的camera实例在onResume的第二次才不是空值？   ---先进一遍流程再实际执行？
3. 在fragment中，使用binding来设置控件操作不行，得用`view.findViewById`再设置，是为什么?
4.

导出组件：android:exported 属性设置为 false什么意思？



## tips
1.
2.
