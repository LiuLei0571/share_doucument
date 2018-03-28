#setting数据是什么？
&emsp;&emsp;setting 数据库管理了系统的一些常用设置，本质是一个系统级别的apk，比如铃声 uri ,音量，本地语言等。用来保证下次用户开机时候和前次开机的配置信息是一致的。也就说明这个数据库是系统级别的数据库，不同的APP只要在权限满足的情况下都是可以调用这个数据库的，可以达到某些数据共享的特性，在Android运行中发挥着很大作用， 用户很难在这方面进行操作。
&emsp;&emsp;数据库的名字是 setting.db ，创建了2个表，分别是system，secure，下文的会具体指出在哪里。
#setting关键类
这里分成2部分，一部分是对外的提供的API，另外一部分是具体的实现。
对外提供的 API 分析是在：
> /frameworks/base/core/java/android/provider/Setting.java

数据库的具体实现是在：

> /frameworks/base/packages/SettingProvider/src/com.android.provides.settings/
 其中的 DatabaseHelper.java 是一个具体的操作类，该类继承了SQLiteOpenHelper,可知道setting的数据库名字为 setting.db
 
 ```
 
 class DatabaseHelper extends SQLiteOpenHelper {
    private static final String TAG = "SettingsProvider";
    private static final String DATABASE_NAME = "settings.db";

    // Please, please please. If you update the database version, check to make sure the
    // database gets upgraded properly. At a minimum, please confirm that 'upgradeVersion'
    // is properly propagated through your change.  Not doing so will result in a loss of user
    // settings.
    private static final int DATABASE_VERSION = 118;

    private Context mContext;
    private int mUserHandle;

    private static final HashSet<String> mValidTables = new HashSet<String>();

    private static final String DATABASE_JOURNAL_SUFFIX = "-journal";
    private static final String DATABASE_BACKUP_SUFFIX = "-backup";

    private static final String TABLE_SYSTEM = "system";
    private static final String TABLE_SECURE = "secure";
    private static final String TABLE_GLOBAL = "global";

    static {
        mValidTables.add(TABLE_SYSTEM);
        mValidTables.add(TABLE_SECURE);
        mValidTables.add(TABLE_GLOBAL);

        // These are old.
        mValidTables.add("bluetooth_devices");
        mValidTables.add("bookmarks");
        mValidTables.add("favorites");
        mValidTables.add("old_favorites");
        mValidTables.add("android_metadata");
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE system (" +
                    "_id INTEGER PRIMARY KEY AUTOINCREMENT," +
                    "name TEXT UNIQUE ON CONFLICT REPLACE," +
                    "value TEXT" +
                    ");");
        db.execSQL("CREATE INDEX systemIndex1 ON system (name);");

        createSecureTable(db);

        // Only create the global table for the singleton 'owner' user
        if (mUserHandle == UserHandle.USER_OWNER) {
            createGlobalTable(db);
        }

        db.execSQL("CREATE TABLE bluetooth_devices (" +
                    "_id INTEGER PRIMARY KEY," +
                    "name TEXT," +
                    "addr TEXT," +
                    "channel INTEGER," +
                    "type INTEGER" +
                    ");");

        db.execSQL("CREATE TABLE bookmarks (" +
                    "_id INTEGER PRIMARY KEY," +
                    "title TEXT," +
                    "folder TEXT," +
                    "intent TEXT," +
                    "shortcut INTEGER," +
                    "ordering INTEGER" +
                    ");");

        db.execSQL("CREATE INDEX bookmarksIndex1 ON bookmarks (folder);");
        db.execSQL("CREATE INDEX bookmarksIndex2 ON bookmarks (shortcut);");

        // Populate bookmarks table with initial bookmarks
        boolean onlyCore = false;
        try {
            onlyCore = IPackageManager.Stub.asInterface(ServiceManager.getService(
                    "package")).isOnlyCoreApps();
        } catch (RemoteException e) {
        }
        if (!onlyCore) {
            loadBookmarks(db);
        }

        // Load initial volume levels into DB
        loadVolumeLevels(db);

        // Load inital settings values
        loadSettings(db);
    }

 ```
 
 其中的 loadSettings(db) 一个核心方法，我们接着看这个函数
 
 
```
    private void loadSettings(SQLiteDatabase db) {
        loadSystemSettings(db);
        loadSecureSettings(db);
        // The global table only exists for the 'owner' user
        if (mUserHandle == UserHandle.USER_OWNER) {
            loadGlobalSettings(db);
        }
    }
```



 由此验证了我们之前所说的会初始化 system，secure 两张表。

  > frameworks/base/packages/SettingProvider/res/values/defaults.xml    
  
  所有的 setting 常量的初始化信息均在这个配置文件中。有4个基本的类型，分别是
 
```
   <bool name="def_dim_screen">true</bool>
    <integer name="def_sleep_timeout">-1</integer>
    <string name="def_airplane_mode_radios" translatable="false">cell,bluetooth,wifi,nfc,wimax</string>
    <string name="airplane_mode_toggleable_radios" translatable="false">bluetooth,wifi,nfc</string>
    <fraction name="def_window_animation_scale">100%</fraction>
```
  
#setting使用场景

    * 系统的一些常量，比如上述说到的WiFi状态，本地语言等
    * 存放一些不能被经常销毁的数据
    * 保存用户设置的值到setting数据库，其他应用或Framework层通过监听SettingsProvider数据库的变化，来做一些相应的处理操作。

#使用注意事项
## 使用方法
### 系统常量的查看与使用
&emsp;我们以无线网络是否可有无线网络通知为例，对应的 key 值为：WIFI_NETWORKS_AVAILABLE_NOTIFICATION_ON，
我们在 settings.java 中确认是否有 WIFI_NETWORKS_AVAILABLE_NOTIFICATION_ON 该变量。
```
Global.WIFI_NETWORKS_AVAILABLE_NOTIFICATION_ON
```
接着我们在 DatabaseHelper.java

```
      loadBooleanSetting(stmt, Settings.Global.WIFI_NETWORKS_AVAILABLE_NOTIFICATION_ON,
                    R.bool.def_networks_available_notification_on);

```

我们又再次发现 Global.WIFI_NETWORKS_AVAILABLE_NOTIFICATION_ON 的值对应的是 R.bool.def_networks_available_notification_on 那么我再去 default.xml中继续找：

```
    <bool name="def_networks_available_notification_on">true</bool>
```
可以看到默认是true的状态。
使用方法

```
        Settings.Global.getInt(RuntimeInfo.context.getContentResolver(),Settings.Global.WIFI_NETWORKS_AVAILABLE_NOTIFICATION_ON,0)!=0);

```

###在setting数据中增加一个自定义的全局常量


     写字符串Settings.System.putString(ContentResolver resolver, String name, String value)

     读字符串Settings.System.getString(ContentResolver resolver, String name)

 

     写整型Settings.System.putInt(ContentResolver resolver, String name, int value)

     读整型Settings.System.getInt(ContentResolver resolver, String name，0)


## 权限申请       
需要文件读写权限

```
<uses-permission android:name="android.permission.READ_SETTINGS" /> 
<uses-permission android:name="android.permission.WRITE_SETTINGS" /> 
```

##setting数据库版本迭代
Android M之前的 SettingsProvider ：数据存放在自建的表当中
Android L的 SettingsProvider ：不在使用数据库来存储系统设置，也就说找不到了setting.db 了，而是通过 xml 将系统的设置存储在了 /data/system/user 目录下 
Android 7.0 之后已经不再使用原来的 DatabaseHelper 来处理数据库了，现在是使用新的SettingProvider.java

