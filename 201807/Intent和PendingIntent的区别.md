#定义
* Intent 是及时启动，Intent随所在的的activity消失而消失
* PendingIntent 是对Intent的封装，用于处理即将发生的事情，，比如 在通知 Notification 中用于跳转界面，但不是立马跳转，一般用作于 Notification


## 具体还是来讲讲PendingIntent
PendingIntent 是一个Intent 的描述，可以把这个描述交给别的程序，别的程序根据后面这个描述所持有的内容在后面的时间做一些其他的事情，实际操作的是对里面的Content参数，并且 PendingIntent 可以取消。
例如，PendingIntent 调用getService 接口，可以包装一个启动的 Service 的 Intent。其他的三大组件也类似。
PendingIntent 有4种 flag：
* FLAG_ONE_SHOT                只执行一次
* FLAG_NO_CREATE               若描述的Intent不存在则返回NULL值
* FLAG_CANCEL_CURRENT          如果描述的PendingIntent已经存在，则在产生新的Intent之前会先取消掉当前的
* FLAG_UPDATE_CURRENT          总是执行,这个flag用的最多



# 区别
* Intent是立即使用的， 而 PendingIntent 可以等到时间发生后触发，PendingIntent 还可以取消。
* Intent 在程序结束后即终止，而 PendingIntent 在程序结束后依然有效
* PendingIntent 自带 Content，而Intent 需要在某个 Content 内运行
* Intent 在原来的task中运行，PendingIntent 在新的 task 中使用


