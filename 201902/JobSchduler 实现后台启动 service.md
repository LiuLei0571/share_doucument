# 背景
在 Android 8.0 以上的系统中，当应用在后台存活 60s 的样子，会被系统标记为”后台应用“，若此时在启动 context.startService() 将会抛出异常。

```
: java.lang.IllegalStateException: Not allowed to start service Intent { [intent 信息] }: app is in background uid UidRecord{6987aa5 u0a358 LAST bg:+1m18s718ms idle procs:2 seq(0,0,0)}
02-28 15:58:24.262 9751-9774/com.getui.yl:pushservice W/System.err:     at android.app.ContextImpl.startServiceCommon(ContextImpl.java:1701)
02-28 15:58:24.262 9751-9774/com.getui.yl:pushservice W/System.err:     at android.app.ContextImpl.startService(ContextImpl.java:1657)
02-28 15:58:24.262 9751-9774/com.getui.yl:pushservice W/System.err:     at android.content.ContextWrapper.startService(ContextWrapper.java:644)
```

# 权限申请


# 注册 JobSchduler

# 接收 JobSchudler

