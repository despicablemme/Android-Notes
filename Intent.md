# Intent
[Google官方文档](https://developer.android.com/guide/components/intents-filters?hl=zh-cn)

[优质文档：intent总结](https://www.jianshu.com/p/ef7b5cd205d2)

当一个activity想要调用另一个activity，它需要设置intent，其中包含了要调用的intent信息，
并且其他activity在manifest中注册了intent-filter，并设置了能够唤起该activity的intent，
因此当请求activity的intent发出，能够相应该intent的activity会做出响应。多个activity均可以响应时
会弹出列表供用户选择。

上述为**隐式**intent，**显式**intent可以直接通过对应activity来调用（已知其信息和怎么调用）。
## 写法
### xml配置接受活动的intent-filter
action代表行为（系统默认或自定义）；类别；data的过滤条件（对uri四个部分的筛选）；
```
<intent-filter
    android:action=""
    android:category=""
    <data
        android:scheme="" android:host=""
        android:port="" android:path="" />
    <mimeType  >
<\intent-filter>
```
**BroadcastReceiver的intent可以在代码中通过`IntentFilter`类来配置。**

### 设置Intent,并发送
Intent中包含的信息：组件名称（想要启动的activity）、操作（想要执行的）、数据（对方执行操作所需）、
类别（LAUNCHER、BROWSARBLE、DEFAULT...）,Extra（额外信息，键值对方式表达）；

#### 自定义intent action
除默认上述信息常量外，需要自定义声明；只要接收方的intent-filter中设置了相同的intent即可接收到；
```
static final String ACTION_LAUGH = "com.example.action.LAUGH";
static final String EXTRA_USERNAME = "com.example.EXTRA_USERNAME";
```
#### 创建一个显式Intent
```
// Executed in an Activity, so 'this' is the Context
// The fileUrl is a string URL, such as "http://www.example.com/image.png"
Intent downloadIntent = new Intent(this, DownloadService.class);
downloadIntent.setData(Uri.parse(fileUrl));
startService(downloadIntent);
```
#### 创建一个隐式Intent
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
**启动service时，应始终指定组件名称（显式启动）**

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

### 接收返回结果
使用`startActivityForResult(intent, int requestCode)`，并重写onActivityResult，前后requestCode要对应:
```
//原 Activity 中启动新 Activity 并请求返回数据
Intent intent = new Intent(this,TagerActivity.class);
startActivityForResult(intent,1);

------------------------------------

//新 Activity 中设定返回 Intent 并销毁，销毁后会回调到原 Activity 的 onActivityResult()方法
Intent intent=new Intent();
intent.putExtra("extra_boolean",true);
setResult(RESULT_OK,intent);
finish();

------------------------------------

//原 Activity 中重写 onActivityResult() 方法
@Override protected void onActivityResult(int requestCode, int resultCode, Intent data) {
switch (requestCode){
  case 1:
    if (resultCode==RESULT_OK){
      boolean b=data.getBooleanExtra("extra_boolean",false);
    }
    break;
  default:
}
}
```

### 自定义类型，序列化数据传送
#### Serializable
Java自带序列化方法，效率较低，类对象直接实现`Serializable`即可:
```
public class MyClass implement Serializable {
    ....
}
```
#### parcelable
Android方法，效率较高，适合在内存中交换数据，binder传输；

```
public class User implements Parcelable {
  ... // 变量和方法

  //步骤1：重写 describeContents()  返回0即可
  @Override public int describeContents() {
    return 0;
  }

  //步骤2：重写 writeToParcel() 写出字段
  @Override public void writeToParcel(Parcel dest, int flags) {
    dest.writeString(。。。);
    dest.writeInt(。。。;
  }

  //步骤3：提供一个 CREATOR 常量
  public static final Parcelable.Creator<User> CREATOR = new Parcelable.Creator<User>() {
    @Override public User createFromParcel(Parcel source) {
      User user = new User();
      user.id = source.readInt();
      user.name = source.readString();
      return user;
    }

    @Override public User[] newArray(int size) {
      return new User[size];
    }
  };
}
```
使用`putExtra()`放入对象，使用`getParcelableExtra()`来取出；

### 非空判断
找不到匹配的intent会调用失败，应用闪退。可用`resolveActivity()`判断是否存在匹配；
```
if (sendIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(sendIntent);
}
```


## pendingintent
一个包装了常规intent的类，由于多做了一层封装，传递出去后不会直接执行，会在对应条件满足后，执行。
```

```
[与intent区别和特点](https://juejin.cn/post/7122767360976486413)
