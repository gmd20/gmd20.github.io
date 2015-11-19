1.创建notification   可以指定 led 灯的开关和颜色
------------------------------------------------
http://developer.android.com/guide/topics/ui/notifiers/notifications.html
http://developer.android.com/reference/android/app/NotificationManager.html#notify(int, android.app.Notification)


2. Updating Notifications  应该只能更新自己应用创建的通知
----------------------------------------------------------
http://developer.android.com/training/notify-user/managing.html


3.NotificationListenerService 可以注册监控 通知的创建的事件。 包含短信，电话，电池等等通知消息，
-----------------------------------------------------------------------------------------------
http://developer.android.com/reference/android/service/notification/NotificationListenerService.html#cancelAllNotifications()

下面这个两个方法应该直接可以在 Listener里面收到通知后，直接调用
这个方法应该可以取消其他应用的通知消息。
  好像代码里面会比较用户名是不是有权限之类的，但好像没有检查application
  name了。其他有的的cancel 的函数会检查application
  name。自能cancel自己应用创建的notification
cancelNotification(String key)

这个方法可以 设置通知被用户查看过了的效果。
setNotificationsShown (String[] keys)


4.相关的原码
------------
https://github.com/android/platform_frameworks_base/blob/master/core/java/android/service/notification/NotificationListenerService.java
https://github.com/android/platform_frameworks_base/blob/d59921149bb5948ffbcb9a9e832e9ac1538e05a0/services/core/java/com/android/server/notification/NotificationManagerService.java


5.总结， 怎么实现 半夜收到短消息不亮led灯的功能。
------------------------------------------------

有个第3放应用 Light Flow
可以修改其他应用的通知的行为，比如led的开关颜色和声音等。不过不知道在新版的
android里面，还能不能用。 andorid 4.3 新加的NotificationListenerService 接口
还说专门是为了方面light flow这些应用而做的。

短消息的notification类别应该对应  notification::CATEGORY_MESSAGE
自己实现NotificationListenerService
接口，应该可以监听所有的notification创建的消息，
读取到所有notification消息的内容。但不清楚NotificationListenerService里面获取到的
Notification能不能被修改，比如修改led 相关fllag。但感觉修改了也没法通知
NotificationManger来更新。


一个可以变通的做法就是，监听到短信的notification之后，就获取消息内容，cancel掉他。
等到早上 屏幕开了，在新建立一个notification 显示消息内容出来。
另外一个是调用setNotificationsShown 消息被用户读取了。
这样应该都可以关闭led，不需要在晚上一直亮着吧。
