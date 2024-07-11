# Broadcast

# 问题记录
1. intent或者pendingIntent发送广播，如果使用的是模糊intent，在broadcast中会无法接收与执行？
- 将intent改为显式intent，携带packadgeContext与`broadcast.class`;
2. 

## 发送广播
`sendBroadcast(Intent intent);`

## 注册receiver

### 静态注册
直接写在manifest里，
```
<application>
    <receiver>
        <intent-filter>...</intent-filter>
    </receiver>
</application>
```
自定义Receiver类继承`BroadcastReceiver`，重写`onReceive(Context, Intent){}`。

### 动态注册
1. 在要接收广播的类里创建一个IntentFilter，加入要接收的过滤项；
```
private static IntentFilter getIntentFilter() {
    final IntentFilter intentFilter = new IntentFilter();
    intentFilter.addAction("android.hardware.usb.action.USB_DEVICE_ATTACHED");
    return intentFilter;
    }
```
2. new一个broadcastReceiver:
```
private BroadcastReceiver receiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        System.out.println("广播接受者："+action);
    }
};
```
3. Activity生命周期内动态注册和动态取消Receiver：
```
@Override
protected void onResume() {
    super.onResume();
        //注册
        registerReceiver(receiver , Filter());
}
@Override
protected void onPause() {
    super.onPause();
      //取消
         unregisterReceiver(receiver);
}
```

## 自定义广播
在receiver定义xml中写入自定义的action字符串：
```
<receiver android:name=".Receiver">  
      <intent-filter>  
              <action android:name="cn.text"/>  
    </intent-filter>  
</receiver>  
```
发送该intent action即可。

## 无序广播
直接发送的广播，随机发送给各个目标；

## 有序广播
按照设定的优先级有依次发送；
```
// 静态设置优先级
<intent-filter
    android:priority="100">
    <action android:name="com.sqw.testBroadcastReceiver.say.hi"/>
    <category android:name="android.intent.category.DEFAULT"/>
</intent-filter>
// 动态设置优先级
IntentFilter intentFilter = new IntentFilter("com.sqw.testBroadcastReceiver.say.hi");
intentFilter.setPriority(100);
```
### 有序广播中断
```
public class MyReceiver2  extends BroadcastReceiver {

    private static final String TAG = "MyReceiver2";

    @Override
    public void onReceive(Context context, Intent intent) {

        Toast.makeText(context,"MyReceiver2 收到广播",Toast.LENGTH_SHORT).show();
        Log.e(TAG, "onReceive: MyReceiver2 收到广播");
        //终止广播像低优先级传递
        abortBroadcast();
    }
}
```
## 本地广播
仅在本应用内发送，安全性高，使用`LocalBroadcastManager`;

发送方式：
```
Intent intent = new Intent("com.sqw.testBroadcastReceiver.say.hi.local");
LocalBroadcastManager.getInstance(context).sendBroadcast(intent);
```
注册与反注册：
```
LocalReceiver1 localReceiver1 = new LocalReceiver1();
IntentFilter intentFilter = new IntentFilter("com.sqw.testBroadcastReceiver.say.hi.local");
//注册本地广播
LocalBroadcastManager.getInstance(context).registerReceiver(localReceiver1,intentFilter);

if(localReceiver1 != null){
    //反注册本地广播
    LocalBroadcastManager.getInstance(context).unregisterReceiver(localReceiver1);
}
```

## 安全性
通过权限限制是否可以接收广播或是否有权限发送广播过来；

### 发送广播时携带权限
只有申请了该权限的才能接收到这个广播;
```
Intent intent = new Intent("com.sqw.testBroadcastReceiver.say.hi");
String receiverPermission = "com.sqw.testBroadcastReceiver.send.RECEIVE_PERMISSION";
sendBroadcast(intent,receiverPermission);          // 无序
sendOrderedBroadcast(intent,receiverPermission);   // 有序
```
自定义权限需要发送方manifest文件中自定义；
`<permission android:name="com.sqw.testBroadcastReceiver.send.RECEIVE_PERMISSION"/>`
接收方则需要申请权限：
`<uses-permission android:name="com.sqw.testBroadcastReceiver.send.RECEIVE_PERMISSION"/>`
应用内发送权限广播，在应用内接收方仍需要申请权限；

## 接收方限制可过来的广播
接收方定义权限、在接收器内使用权限，则发送方必须要在manifest申请了权限，才可发送至该接收:
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    ...
    <permission android:name="com.sqw.testBroadcastReceiver.receive.RECEIVE_PERMISSION"/>

    <application
        ...
        <receiver android:name=".receiver.ServiceReceiver"
            android:permission="com.sqw.testBroadcastReceiver.receive.RECEIVE_PERMISSION">
            ...
        </receiver>
    </application>
</manifest>
```
发送方申请该权限：
```
<uses-permission android:name="com.sqw.testBroadcastReceiver.receive.RECEIVE_PERMISSION"/>
```
也可以双向鉴权。

## 参考（摘抄）
1. 如果您不需要向应用以外的组件发送广播，则可以使用支持库中提供的 LocalBroadcastManager 来收发本地广播。LocalBroadcastManager 效率更高（无需进行进程间通信），并且您无需考虑其他应用在收发您的广播时带来的任何安全问题。本地广播可在您的应用中作为通用的发布/订阅事件总线，而不会产生任何系统级广播开销。

2. 如果有许多应用在其清单中注册接收相同的广播，可能会导致系统启动大量应用，从而对设备性能和用户体验造成严重影响。为避免发生这种情况，请优先使用上下文注册而不是清单声明。有时，Android 系统本身会强制使用上下文注册的接收器。例如，CONNECTIVITY_ACTION 广播只会传送给上下文注册的接收器。

3. 请勿使用隐式 intent 广播敏感信息。任何注册接收广播的应用都可以读取这些信息。您可以通过以下三种方式控制哪些应用可以接收您的广播：

4. 您可以在发送广播时指定权限。
在 Android 4.0 及更高版本中，您可以在发送广播时使用 setPackage(String) 指定软件包。系统会将广播限定到与该软件包匹配的一组应用。
您可以使用 LocalBroadcastManager 发送本地广播。
当您注册接收器时，任何应用都可以向您应用的接收器发送潜在的恶意广播。您可以通过以下三种方式限制您的应用可以接收的广播：

5. 您可以在注册广播接收器时指定权限。
对于清单声明的接收器，您可以在清单中将 android:exported 属性设置为“false”。这样一来，接收器就不会接收来自应用外部的广播。
您可以使用 LocalBroadcastManager 限制您的应用只接收本地广播。
广播操作的命名空间是全局性的。请确保在您自己的命名空间中编写操作名称和其他字符串，否则可能会无意中与其他应用发生冲突。

6. 由于接收器的 onReceive(Context, Intent) 方法在主线程上运行，因此它会快速执行并返回。如果您需要执行长时间运行的工作，请谨慎生成线程或启动后台服务，因为系统可能会在 onReceive() 返回后终止整个进程。如需了解详情，请参阅对进程状态的影响。要执行长时间运行的工作，我们建议：

7. 在接收器的 onReceive() 方法中调用 goAsync()，并将 BroadcastReceiver.PendingResult 传递给后台线程。这样，在从 onReceive() 返回后，广播仍可保持活跃状态。不过，即使采用这种方法，系统仍希望您非常快速地完成广播（在 10 秒以内）。为避免影响主线程，它允许您将工作移到另一个线程。
使用 JobScheduler 调度作业。如需了解详情，请参阅智能作业调度。
请勿从广播接收器启动 Activity，否则会影响用户体验，尤其是有多个接收器时。相反，可以考虑显示通知。

[参考链接](https://www.jianshu.com/p/b4bc91afa85d)
