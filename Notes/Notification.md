# Notification

> Notification在我们生活中或多或少大家都接触到。Notification 可以让我们在获得消息的时候，在状态栏、锁屏的界面来显示相应的信息。如果说没有了Notification，那么我们的QQ和微信以及其他应用都没法主动通知我们。

> 此处简单说一下3种Notification。分别是：普通Notification、折叠式Notification和悬挂式Notification。

* 普通Notification

首先，创建Builder对象，用PendingIntent控制跳转，这里跳转网页。

```
Notification.Builder builder = new Notification.Builder(this);
        Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse("https://github.com/JiaYang627"));
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, intent, 0);
```

接着通过builder，我们就可以给Notification添加各种属性了：

```
builder.setContentIntent(pendingIntent);
        builder.setSmallIcon(R.mipmap.ic_launcher);
        builder.setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher));
        builder.setAutoCancel(true);
        builder.setContentTitle("普通通知");
```

* 折叠式Notification

> 折叠式Notification 是一种自定义视图的Notification，用来显示长文本和一些自定义布局的场景。

> 它有两种场景：一种就是普通状态下的视图(如果不是自定义的话，和上面的Notification的视图场景样式一样)。另外一种：展开状态下的视图，和普通Notification不用的是，我们需要自定义视图，而这个视图显示的进程和我们创建视图的进程不在一个进程，所以我们需要使用RemoteViews。

* 首先 使用RemoteViews创建我们的自定义视图

```
RemoteViews remoteViews = new RemoteViews(getPackageName(), R.layout.view_fold);
```

然后我们需要把我们自定义的视图赋值给Notification的视图：

```
Notification build = builder.build();
        build.bigContentView = remoteViews;
```
当然我们也可以将自定义的视图设置为Notification普通状态下的视图：

```
Notification build = builder.build();
        build.contentView = remoteViews;
```


* 悬挂式Notification

> 悬挂式Notification是Android5.0新增加的方式。不用于前两种的是：悬挂式Notification不需要下拉通知栏就直接显示出来，悬挂在屏幕上方，并且焦点不变，仍在用户操作的界面，不会打断用户的操作。过几秒后就会自动消失。但是，它需要调用setFullScreenIntet来将Notification变为悬挂式Notification。


```
PendingIntent hangPendingIntent = PendingIntent.getActivity(this, 0, hangIntent, PendingIntent.FLAG_CANCEL_CURRENT);
        builder.setFullScreenIntent(hangPendingIntent, true);
```

* Notification的显示等级

Android5.0加入了一种新的模式Notification的显示等级，有一下3种：
* VISIBILITY_PUBLIC：任何情况下都会显示通知。
* VISIBILITY_PRIVATE：只有在没有锁屏的时候会显示通知。
* VISIBILITY_SECRET：在pin、password等安全锁和没有锁屏的情况下才能够显示通知。

设置也非常简单，只要调用setVisibility方法就可以调用了：

```
builder.setVisibility(Notification.VISIBILITY_PUBLIC);
```


具体Demo:
* [GitHubNotesDemo](https://github.com/JiaYang627/GitHubNotesDemo)

