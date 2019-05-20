# 概述
从字面上可以得出是远程视图的意思，但是该视图不是当前进程的View，而是属于SystemServer进程. 应用程序与RemoteViews之间依赖Binder实现了进程间通信.
一般用作于在桌面小程序或者通知栏的自定义布局.
# 应用
## 在通知栏上的使用
当需要对通知栏进行自定义布局的时候，会用到 RemoteView

```
  notification.contentView = new RemoteViews(CoreRuntimeInfo.pkgName, resIDlayout);

 notification.contentIntent = getPendingIntent(taskid, messageid, notifID, notificationBean, false);
  NotificationShow.showNotification(notificationManager, notifID, notification, 4);
```
使用方法非常的简单，直接 new 一个就就完事了，因为 RemoteView 是在单独的进程，所以我们其实是拿不到 view 的 ID 的，需要使用 RemoteView 提供的方法来更新 view ，比如 
```
remoteView.setImageViewResource(int ID，int drawable);
```
## 桌面小控件
### 定义布局文件
定义一个叫 widget.xml 的布局文件
```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.zsxq.liulei.remoteview">

    <application>
        <receiver android:name=".MyAppWidgetProvider">
            <meta-data
                android:name="android.appwidget.provider"
                android:resource="@layout/widget" />
            <intent-filter>
                <action android:name="com.lenny.click" />
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
            </intent-filter>
        </receiver>
    </application>
</manifest>

```

### 定义小部件配置信息
在 res/xml 中定义小控件的初始属性
```
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:initialLayout="@layout/widget"
    android:minHeight="50dp"
    android:minWidth="50dp"
    android:updatePeriodMillis="8640000">

</appwidget-provider>
```
android:initialLayout 是指小工具使用的初始化布局。
android:updatePeriodMillis 是自动更新的时间，单位是毫秒

## 小控件的具体实现类

```

public class MyAppWidgetProvider extends AppWidgetProvider {
    //分发具体的事件给其他方法
    @Override
    public void onReceive(Context context, Intent intent) {
        super.onReceive(context, intent);
        AppWidgetManager appWidgetManager = AppWidgetManager.getInstance(context);
        RemoteViews remoteView = new RemoteViews(context.getPackageName(), R.layout.widget);
        Intent intent1 = new Intent();
        intent1.setAction("com.lenny.click");
        PendingIntent pendingIntent = PendingIntent.getBroadcast(context, 0, intent1, 0);
        //remoteView 的点击事件需要如下处理
        remoteView.setOnClickPendingIntent(R.id.iv_widget, pendingIntent);
        appWidgetManager.updateAppWidget(new ComponentName(context, MyAppWidgetProvider.class), remoteView);
    }

    //每次桌面小部件更新都会调用此方法
    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        super.onUpdate(context, appWidgetManager, appWidgetIds);
    }
    //第一次添加到桌面会被调用
    @Override
    public void onEnabled(Context context) {
        super.onEnabled(context);
    }
    //删除控件会被调用
    @Override
    public void onDeleted(Context context, int[] appWidgetIds) {
        super.onDeleted(context, appWidgetIds);
    }
    //当最后一个该类型的小部件被删除时，会被调用
    @Override
    public void onDisabled(Context context) {
        super.onDisabled(context);
    }
}

```
### 声明小部件

```
  <receiver android:name=".MyAppWidgetProvider">
            <meta-data
                android:name="android.appwidget.provider"
                android:resource="@layout/widget" />
            <intent-filter>
                <action android:name="com.lenny.click" />
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
            </intent-filter>
        </receiver>
    </application>
```