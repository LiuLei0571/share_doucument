1 App Links
    通过http（https）地址打开App。app先注册的关联host，当用户点击链接时，系统解析host地址，与app注册的信息匹配，如匹配成功则直接打开APP，匹配失败则打开浏览器。
    例如，若App想通过 http://geyan.getui.com 打开，则可按如下步骤实现。
    (1) 在manifest中声明处理App Links的Activity，其中Intent Filter必须包含：
        (a) 必须包含category：android.intent.action.VIEW and android.intent.category.BROWSABLE
        (b) 必须声明android:scheme为 http／https
        完整版如下：
        <activity ...>
        <intent-filter  android:autoVerify="true">
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
            <category android:name="android.intent.category.BROWSABLE" />
            <data android:scheme="http" />
            <data android:scheme="https" />
            <data android:host="geyan.getui.com" />
        </intent-filter>
        </activity>
    (2) 在host服务器的geyan.getui.com/.well-known的路径下，添加配置 assetlinks.json。
        (a) 完整路径：https://geyan.getui.com/.well-known/assetlinks.json。
        (b) assetlinks.json内容
            [{
            "relation": ["delegate_permission/common.handle_all_urls"],
            "target": {
                "namespace": "android_app",
                "package_name": "com.example",
                "sha256_cert_fingerprints":
                ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
            }
            }]
            其中：
                package_name：为应用包名
                sha256_cert_fingerprints：通过 keytool -list -v -keystore my-release-key.keystore 查看
        (c) 若一个host需要配置多个app，assetlinks.json添加多个app的信息。
        (d) 若一个app需要配置多个host，每个host的.well-known下都要配置assetlinks.json
    (3) 目标Activity如何处理
        Intent intent = getIntent();
        Uri applink = intent.getData();
        applink的值为url链接
    
2 机型适配情况    
    链接为：http://geyan.getui.com
    6.0以上可以直接打开应用；6.0以下会弹出应用选择框，选择目标应用。
    各个应用行为如下：
    小米mix（Android 6.0）
        短信：点击链接，弹出框，选择访问，可以直接跳转倒App。
        联系人：网站条目添加链接，点击直接跳转到App
        Outlook客户端：点击链接，直接跳转到App
        自带邮箱客户端：点击链接，直接跳转到App
        网易邮箱：点击链接进入webview，点击右上角浏览器打开，进入App
        QQ：在聊天界面，点击链接会跳转到web页面，在点击右上角用浏览器打开，直接跳转到应用
        微信：点击链接会跳转到web页面，在点击右上角用浏览器打开，直接跳转到应用
        微博：点击链接会跳转到web页面，在点击右上角用浏览器打开，只能到浏览器
        java代码：
            Uri uri = Uri.parse("http://geyan.getui.com/ggg");
            startActivity(new Intent(Intent.ACTION_VIEW,uri));
    三星 A8（Android 5.0）
        短信：点击链接，弹出框，选择打开链接，弹出多个应用选择。
        联系人：网站条目添加链接，点击会弹出多个应用选择。
        QQ：在聊天界面，点击链接会跳转到web页面，在点击右上角用浏览器打开，会弹出多个应用选择
        outlook邮箱：点击链接，会弹出多个应用选择
        微博：点击链接会跳转到web页面，在点击右上角用浏览器打开，只能到浏览器
        java代码：会弹出多个应用选择
    索尼z2（Android 4.4）
        短信：点击链接，弹出框，选择打开链接，会弹出多个应用选择。
        联系人：网站条目添加链接，点击会弹出多个应用选择。
        QQ：在聊天界面，点击链接会跳转到web页面，在点击右上角用浏览器打开，会弹出多个应用选择
        网易邮箱：点击进入webview，点击右上角浏览器打开，会弹出多个应用选择
        微博：点击链接会跳转到web页面，在点击右上角用浏览器打开，只能到浏览器
        java代码：会弹出多个应用选择

