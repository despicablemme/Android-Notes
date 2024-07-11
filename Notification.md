# Notofication
应用通知

## 通知生成方法
1. 设置通知channel，传入一个独一无二的channelId；
2. 如需自定义通知样式，new一个RemoteViews实例；
3. 设置intent，可设置点击通知项的pendingIntent，或者自定义view有按钮的话设置pendingIntent至对应的子view id;
pendingInent必须设置flag参数，MUTABLE或者相反（是否可更改）；
4. 通过builder构建一个通知，并notify发送，`notify(...)`第一个参数id独特代表这个通知；

下面代码在通知中自定义了两个按键：
5. `setStyle`可以设置大图模式、大文字模式等。。。
```
private void sendCustomNewCaptureNotice() {

        NotificationChannel notificationChannel = new NotificationChannel("cap", "capture",
                NotificationManager.IMPORTANCE_HIGH);
        NotificationManagerCompat notificationManager = NotificationManagerCompat.from(this);
        notificationManager.createNotificationChannel(notificationChannel);

        RemoteViews remoteViews = new RemoteViews(getPackageName(), R.layout.notification_layout);

        Intent intent = new Intent(this, PicViewActivity.class);
        intent.setAction(Intent.ACTION_VIEW);
        intent.setData(curFileUri);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, intent, PendingIntent.FLAG_IMMUTABLE);
        remoteViews.setOnClickPendingIntent(R.id.notify_open_button, pendingIntent);

        Intent intentDelete = new Intent(ACTION_DELETE);
        intent.setData(curFileUri);
        PendingIntent pendingIntentDelete = PendingIntent.getBroadcast(this, 0, intentDelete, PendingIntent.FLAG_IMMUTABLE);
        remoteViews.setOnClickPendingIntent(R.id.notify_delete_button, pendingIntentDelete);

        NotificationCompat.Builder builder = new NotificationCompat.Builder(this, "cap")
                .setContentTitle("new capture!")
                .setContentInfo("Picture available, click and check details!")
                .setLargeIcon(BitmapFactory.decodeFile(getPathFromUri(curFileUri)))
                .setSmallIcon(R.mipmap.ic_launcher)
                // .setContentIntent(pendingIntent)
                .setStyle(new NotificationCompat.DecoratedCustomViewStyle())
                .setCustomContentView(remoteViews)
                .setAutoCancel(true);
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.POST_NOTIFICATIONS) != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.POST_NOTIFICATIONS}, 234);
        }
        notificationManager.notify(nameNumber, builder.build());
        nameNumber += 1;

    }
```

## 自定义通知操作（点击等）
如上代码在通知中设置两个按键，设置对应pendingIntent，来实现点击按键时启动对应activity或发送广播，
在广播receiver的`onReceive`内，可以通过得到`NotificationManager`且通过intent传递的notifyId来cancel
对应的notification。
```
public void onReceive(Context context, Intent intent) {
        Log.d(TAG, "onReceive: broadcast receicved");
        context.getContentResolver().delete(intent.getData(), null);
        Log.d(TAG, "onReceive: deleted, then delete the notice");
        // intent.getComponent()
        NotificationManagerCompat manager = NotificationManagerCompat.from(context);
        // manager.deleteNotificationChannel("cap");
        int notificationId = intent.getIntExtra("notificationId", 0);
        manager.cancel(notificationId);
    }
```

[所谓"看这篇就够了"](https://cloud.tencent.com/developer/article/2031617)

[看这篇也够了](https://blog.csdn.net/shanshui911587154/article/details/105683683)
