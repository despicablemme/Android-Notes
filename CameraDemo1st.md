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

## Activity
[Google官方文档](https://developer.android.com/guide/components/activities/intro-activities?index=..%2F..%2Fandroid-training&hl=zh-cn#0)

### activity的生命周期
[Google](https://developer.android.com/guide/components/activities/activity-lifecycle?hl=zh-cn)

### activity重启接力棒 savedInstanceBundle
将本次activity关闭后的一些必要数据保存，下次create可以直接显示出来。
```
TextView textView;

// Some transient state for the activity instance.
String gameState;

@Override
public void onCreate(Bundle savedInstanceState) {
    // Call the superclass onCreate to complete the creation of
    // the activity, like the view hierarchy.
    super.onCreate(savedInstanceState);

    // Recover the instance state.
    if (savedInstanceState != null) {
        gameState = savedInstanceState.getString(GAME_STATE_KEY);
    }

    // Set the user interface layout for this activity.
    // The layout is defined in the project res/layout/main_activity.xml file.
    setContentView(R.layout.main_activity);

    // Initialize member TextView so it is available later.
    textView = (TextView) findViewById(R.id.text_view);
}

// This callback is called only when there is a saved instance previously saved using
// onSaveInstanceState(). Some state is restored in onCreate(). Other state can optionally
// be restored here, possibly usable after onStart() has completed.
// The savedInstanceState Bundle is same as the one used in onCreate().
@Override
public void onRestoreInstanceState(Bundle savedInstanceState) {
    textView.setText(savedInstanceState.getString(TEXT_VIEW_KEY));
}

// Invoked when the activity might be temporarily destroyed; save the instance state here.
@Override
public void onSaveInstanceState(Bundle outState) {
    outState.putString(GAME_STATE_KEY, gameState);
    outState.putString(TEXT_VIEW_KEY, textView.getText());

    // Call superclass to save any view hierarchy.
    super.onSaveInstanceState(outState);
}
```
`onSaveInstanceState`的几个注意事项：
1.在一个activity被销毁前，不一定会调用onSaveInstanceState()这个方法，因为不是所有情况都需要去存储activity的状态（例如当用户按回退键退出你的activity的时候，因为用户指定关掉这个activity）。
2.如果这个方法被调用，它一定会在 onStop()方法之前，可能会在onPause()方法之前。
3.布局中的每一个View默认实现了onSaveInstanceState()方法，这样的话，这个UI的任何改变都会自动的存储和在activity重新创建的时候自动的恢复。但是这种情况只有在你为这个UI提供了唯一的ID之后才起作用，如果没有提供ID，将不会存储它的状态。
4.由于默认的onSaveInstanceState()方法的实现帮助UI存储它的状态，所以如果你需要覆盖这个方法去存储额外的状态信息时，你应该在执行任何代码之前都调用父类的onSaveInstanceState()方法（super.onSaveInstanceState()）。
5.由于onSaveInstanceState()方法调用的不确定性，你应该只使用这个方法去记录activity的瞬间状态（UI的状态）。不应该用这个方法去存储持久化数据。当用户离开这个activity的时候应该在onPause()方法中存储持久化数据（例如应该被存储到数据库中的数据）。

### 其他
AppCompatActivity  ---  向下兼容带有新特性的activity基类。extends FragmentActivity,多了一个AppCompatDelegate

`import androidx.core.app.ActivityCompat;`activity辅助类，权限处理等。

## 生命周期感知型组件
[Google](https://developer.android.com/topic/libraries/architecture/lifecycle?hl=zh-cn)

## Intent
[Google官方文档](https://developer.android.com/guide/components/intents-filters?hl=zh-cn)

[优质文档：intent总结](https://www.jianshu.com/p/ef7b5cd205d2)

当一个activity想要调用另一个activity，它需要设置intent，其中包含了要调用的intent信息，
并且其他activity在manifest中注册了intent-filter，并设置了能够唤起该activity的intent，
因此当请求activity的intent发出，能够相应该intent的activity会做出响应。多个activity均可以响应时
会弹出列表供用户选择。

上述为**隐式**intent，**显式**intent可以直接通过对应activity来调用（已知其信息和怎么调用）。

### 配置activity的intent-filter
```
<intent-filter
    android:action=""
    android:category=""
<\intent-filter>
```

### 设置Intent,并调用对应activity
Intent中包含的信息：组件名称（想要启动的activity）、操作（想要执行的）、数据（对方执行操作所需）、
类别（LAUNCHER、BROWSARBLE、DEFAULT...）,Extra（额外信息，键值对方式表达）；

除默认上述信息常量外，需要自定义声明；
```
static final String ACTION_LAUGH = "com.example.action.LAUGH";
static final String EXTRA_USERNAME = "com.example.EXTRA_USERNAME";
```
创建一个显式Intent
```
// Executed in an Activity, so 'this' is the Context
// The fileUrl is a string URL, such as "http://www.example.com/image.png"
Intent downloadIntent = new Intent(this, DownloadService.class);
downloadIntent.setData(Uri.parse(fileUrl));
startService(downloadIntent);
```
创建一个隐式Intent
```
// Create the text message with a string.
Intent sendIntent = new Intent();
sendIntent.setAction(Intent.ACTION_SEND);
sendIntent.putExtra(Intent.EXTRA_TEXT, textMessage);
sendIntent.setType("text/plain");

// intent可能找不到能够执行该需求的activity
try {
    startActivity(sendIntent);
} catch (ActivityNotFoundException e) {
    // Define what your app should do if no activity can handle the intent.
}
```

Manifest中注册可接受的intent的过滤器;若要接受隐式，需要将category设置为DEFAULT；
```
<activity android:name="ShareActivity" android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="text/plain"/>
    </intent-filter>
</activity>
```
### pendingintent
待定Intent

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
3. android activity权限相关？
4. context上下文是什么？

导出组件：android:exported 属性设置为 false什么意思？



## tips
1.
2.
