# 什么是 RemoteViews
RemoteViews 直译过来是远程视图，是属于 SystemServer 进程，应用程序与 RemoteViews 之间依赖 Binder 实现进程间的通信。
# 使用场景
主要是用在通知栏和桌面小插件。
#用法
## 通知栏应用
1. 创建一个应用通知
2. 创建RemoteViews并且设置 Notification 的 RemoteViews
只需要提供当前应用的包名和布局文件的资源id即可创建一个 RemoteViews 对象

```
RemoteViews mRemoteViews=new RemoteViews("com.example.remoteviewdemo", R.layout.remoteview_layout);

```
然后通过 Notification 的 setContent 属性设置 RemoteViews

3.改变 RemoteViews 的布局
    RemoViews 并不能直接获得控件实例，需要传入控件的 id 和响应的修改内容才行
    而且所操作的View是有一定限制的，仅限于
    
Layout：
    FrameLayout、LinearLayout、RelativeLayout、GridLayout
View：
    AnalogClock、Button、Chronometer、ImageButton、ImageView、ProgressBar、TextView、ViewFlipper、ListView、GridView、StackView、AdapterViewFlipper、ViewStub    
    
    
常用的设置方法如下：
  
```
 - setTextViewText(viewId, text)                     设置文本
 - setTextColor(viewId, color)                       设置文本颜色
 - setTextViewTextSize(viewId, units, size)          设置文本大小 
 - setImageViewBitmap(viewId, bitmap)                设置图片
 - setImageViewResource(viewId, srcId)               根据图片资源设置图片
 - setViewPadding(viewId, left, top, right, bottom)  设置Padding间距
 - setOnClickPendingIntent(viewId, pendingIntent)    设置点击事件 

```
##桌面小组件

### 设置桌面小主键的布局
### 编写桌面小组件的配置文件
在res文件夹下新建一个 xml 文件夹并且创建配置文件

```
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:initialLayout="@layout/widget_main"
    android:minHeight="100.0dip"
    android:minWidth="150.0dip"
    android:updatePeriodMillis="8640000">

</appwidget-provider
```
其中部分参数所代表的含义如下

```
initialLayout：桌面小组件的布局XML文件
minHeight：桌面小组件的最小显示高度
minWidth：桌面小组件的最小显示宽度
updatePeriodMillis：桌面小组件的更新周期。这个周期最短是30分钟
```
### 创建桌面小组件更新的 Service

```
public class TimerService extends Service {
    private Timer timer;
    private SimpleDateFormat formatter = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");

    @Override
    public void onCreate() {
        super.onCreate();
        timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                updateViews();
            }
        }, 0, 1000);
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    private void updateViews() {
        String time = formatter.format(new Date());
        RemoteViews remoteView = new RemoteViews(getPackageName(), R.layout.widget_main);
        remoteView.setTextViewText(R.id.widget_main_tv_time, time);
        AppWidgetManager manager = AppWidgetManager.getInstance(getApplicationContext());
        ComponentName componentName = new ComponentName(getApplicationContext(), WidgetProvider.class);
        manager.updateAppWidget(componentName, remoteView);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        timer = null;
    }
}
 

```

### 创建桌面小组件的控制类 APPWidgetProvider

```
import android.appwidget.AppWidgetManager;
import android.appwidget.AppWidgetProvider;
import android.content.Context;
import android.content.Intent;

/**
 * AppWidgetProvider的子类，相当于一个广播
 */
public class WidgetProvider extends AppWidgetProvider {
    /**
     * 当小组件被添加到屏幕上时回调
     */
    @Override
    public void onEnabled(Context context) {
        super.onEnabled(context);
        context.startService(new Intent(context, TimerService.class));
    }

    /**
     * 当小组件被刷新时回调
     */
    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        super.onUpdate(context, appWidgetManager, appWidgetIds);
    }
    /**
     * 当widget小组件从屏幕移除时回调
     */
    @Override
    public void onDeleted(Context context, int[] appWidgetIds) {
        super.onDeleted(context, appWidgetIds);
    }

    /**
     * 当最后一个小组件被从屏幕中移除时回调
     */
    @Override
    public void onDisabled(Context context) {
        super.onDisabled(context);
        context.stopService(new Intent(context, TimerService.class));
    }
}
```
### 配置文件
注册 Service 和 BroadcastReceiver

```
<service android:name=".TimerService" />
<receiver android:name=".WidgetProvider">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/widget" />
</receiver>
```


##原理
  我们的通知和桌面小组件分别是由 NotificationManager 和 APPWidgetManager 管理的，而 NotificationManager 和 APPWidgetManager 又都是通 Binder 分别和 SystemServer 进程中的 NotificationManagerServer 和 APPWidgetServer 进行通信。
  RemoteViews 实现的是 Parcelable 接口，因此，此对象是可以在进程间进行通信的，首先，RemoteViews 通过 Binder 传递到 SystemServer 进程中，系统会根据 RemoteViews 提供的包名，去项目包中找到 RemoteViews 显示的布局资源等。接着根据资源 id 创建布局文件，然后我们调动 set 方法后，通过 NotificationManager 和 APPWidgetManager 来提交更新任务，最后在 ServiceServer 进程中进行具体的操作。


