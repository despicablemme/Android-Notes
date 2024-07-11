# Intent
[Google官方文档](https://developer.android.com/guide/components/intents-filters?hl=zh-cn)

[优质文档：intent总结](https://www.jianshu.com/p/ef7b5cd205d2)

当一个activity想要调用另一个activity，它需要设置intent，其中包含了要调用的intent信息，
并且其他activity在manifest中注册了intent-filter，并设置了能够唤起该activity的intent，
因此当请求activity的intent发出，能够相应该intent的activity会做出响应。多个activity均可以响应时
会弹出列表供用户选择。

上述为**隐式**intent，**显式**intent可以直接通过对应activity来调用（已知其信息和怎么调用）。

## 配置activity的intent-filter
```
<intent-filter
    android:action=""
    android:category=""
<\intent-filter>
```

## 设置Intent,并调用对应activity
Intent中包含的信息：组件名称（想要启动的activity）、操作（想要执行的）、数据（对方执行操作所需）、
类别（LAUNCHER、BROWSARBLE、DEFAULT...）,Extra（额外信息，键值对方式表达）；

除默认上述信息常量外，需要自定义声明；只要接收方的intent-filter中设置了相同的intent即可接收到；
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

## pendingintent
[与intent区别和特点](https://juejin.cn/post/7122767360976486413)
