Android 原生通知推送兼容 8.0
Android 8.0 通知需要设置通知渠道才能正常显示, 步骤如下：

官方创建通知文档：https://developer.android.google.cn/training/notify-user/build-notification

### 定义通知 id、通知渠道 id、通知渠道名

```java
 private static final int PUSH_NOTIFICATION_ID = (0x001);
 private static final String PUSH_CHANNEL_ID = "PUSH_NOTIFY_ID";
 private static final String PUSH_CHANNEL_NAME = "PUSH_NOTIFY_NAME";
```

注：a.通知渠道 id 不能太长且在当前应用中是唯一的, 太长会被截取. 官方说明：
The id of the channel. Must be unique per package. The value may be truncated if it is too long.

b.通知渠道名也不能太长推荐 40 个字符, 太长同样会被截取. 官方说明：
The recommended maximum length is 40 characters; the value may be truncated if it is too long.

### 创建通知渠道 Android8.0

```java
 NotificationManager notificationManager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(PUSH_CHANNEL_ID, PUSH_CHANNEL_NAME, NotificationManager.IMPORTANCE_HIGH);
            if (notificationManager != null) {
                notificationManager.createNotificationChannel(channel);
            }
        }
```

### 发送通知标题 , 内容并显示

```java
 NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
        Intent notificationIntent = new Intent(this, MainActivity.class);
        notificationIntent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, 0);
        builder.setContentTitle("通知标题")//设置通知栏标题
                .setContentIntent(pendingIntent) //设置通知栏点击意图
                .setContentText("通知内容")
                .setTicker("通知内容") //通知首次出现在通知栏, 带上升动画效果的
                .setWhen(System.currentTimeMillis())//通知产生的时间, 会在通知信息里显示, 一般是系统获取到的时间
                .setSmallIcon(R.mipmap.ic_launcher)//设置通知小ICON
                .setChannelId(PUSH_CHANNEL_ID)
                .setDefaults(Notification.DEFAULT_ALL);

        Notification notification = builder.build();
        notification.flags |= Notification.FLAG_AUTO_CANCEL;
        if (notificationManager != null) {
            notificationManager.notify(PUSH_NOTIFICATION_ID, notification);
        }
```

注：setChannelId(PUSH_CHANNEL_ID) 不要遗漏了

案例：SystemUI/src/com/android/systemui/usb/StorageNotification.java

原生代码：

```java
private void onDiskScannedInternal(DiskInfo disk, int volumeCount) {
        if (volumeCount == 0 && disk.size > 0) {
            // No supported volumes found, give user option to format
            final CharSequence title = mContext.getString(
                    R.string.ext_media_unsupported_notification_title, disk.getDescription());
            final CharSequence text = mContext.getString(
                    R.string.ext_media_unsupported_notification_message, disk.getDescription());

            Notification.Builder builder =
                    new Notification.Builder(mContext, NotificationChannels.STORAGE)
                            .setSmallIcon(getSmallIcon(disk, VolumeInfo.STATE_UNMOUNTABLE))
                            .setColor(mContext.getColor(R.color.system_notification_accent_color))
                            .setContentTitle(title)
                            .setContentText(text)
                            .setContentIntent(buildInitPendingIntent(disk))
                            .setStyle(new Notification.BigTextStyle().bigText(text))
                            .setVisibility(Notification.VISIBILITY_PUBLIC)
                            .setLocalOnly(true)
                            .setCategory(Notification.CATEGORY_ERROR)
                            .extend(new Notification.TvExtender());
            SystemUI.overrideNotificationAppName(mContext, builder);
            mNotificationManager.notifyAsUser(disk.getId(), SystemMessage.NOTE_STORAGE_DISK,
                    builder.build(), UserHandle.ALL);
        } else {
            // Yay, we have volumes!
            mNotificationManager.cancelAsUser(disk.getId(), SystemMessage.NOTE_STORAGE_DISK,
                    UserHandle.ALL);
        }
    }
```

以上方法是监听 sdcard 插入的通知处理, 可是在 Android 8.0 该方法的通知无效！

修改成以下方式即可：

```java
NotificationCompat.Builder builder =
            		new NotificationCompat.Builder(mContext);
            builder.setSmallIcon(getSmallIcon(disk, VolumeInfo.STATE_UNMOUNTABLE))
                            .setColor(mContext.getColor(R.color.system_notification_accent_color))
                            .setContentTitle(title)
                            .setContentText(text)
                            .setContentIntent(buildInitPendingIntent(disk))
                            .setVisibility(Notification.VISIBILITY_PUBLIC)
                            .setChannelId(PUSH_CHANNEL_ID)
                            .setCategory(Notification.CATEGORY_ERROR);
           // SystemUI.overrideNotificationAppName(mContext, builder);

            mNotificationManager.notifyAsUser(disk.getId(), SystemMessage.NOTE_STORAGE_DISK,
                    builder.build(), UserHandle.ALL);
```

可在上面代码后面加入通知自动消失处理：

```java
Handler handler = new Handler();
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
               /**
                *要执行的操作
                */
nm.cancel(notifyTag, notifyId);//按tag id 来清除消息
            }
        }, 3000);//3秒后执行Runnable中的run方法
```
