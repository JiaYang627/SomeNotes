# AndroidLaunchMode

> 我们都知道，开启一个Activity，系统会在任务栈中创建实例，而任务栈是一种“后进先出”的栈结构。默认情况下，开启一个Activity，系统会创建实例，如果多次创建同一个实例，难道还要栈中创建多个相同的实例 ？ 当然 是不可以的。由此，Android在设计的时候提供了启动模式来修改系统的默认行为。

* **standard**

```
标准模式，也是系统的默认模式。每次启动一个Act的时候都会创建一个新的实例，系统不管这个实例是否已经存在。

例如：Activity A 启动了Activity B (标准模式)，此时在Act A的栈中 就会新建一个Act B 的实例，此时栈中  Act A 在栈底 Act B在栈顶。
```

* **singleTop**

```
栈顶复用模式。设置这种启动模式的Act，如果此时已经在任务栈的栈顶，那么此Act不会被重新创建，而是会走此Act的onNewIntent，此Act的onCreate、onStart不会被创建。如果此Act在任务栈中已经有实例但是不是在任务栈的栈顶，那此Act仍会重新创建。

例如：当前任务栈中有 ABCD 4个Act，A在栈底，D在栈顶，其中BD的启动模式为singleTop，此时如果在开启Activity D，那么任务栈中情况为：ABCD，如果此时开启Activity B，任务栈中情况为：ABCDB。
```

* **singleTask**

```
栈内复用模式。这是一种单实例模式。设置此启动模式的Act，只要在所在的栈中出现，再多次启动的时候不会重新创建，而是重走onNewIntent方法。

例如(考虑在同一栈中):此时任务栈中有ABC三个Act，A为栈底，B为singleTask，C为栈顶，此时要开启一个ssingleTask的Act D，那么此时就会创建新的Act D。此时任务栈内情况为：ABCD。如果此时再次开启Act B，因为Act B为singleTask，那么此时任务栈内情况为：AB。Act B会重走onNewIntet方法。这里说一下：singleTask默认具有clearTop效果。所以再次开启singleTask的Act B的时候会把Act B上的 C D 销毁。
```

* **singleInstance**

```
单实例模式。一种加强的singleTask模式，这种模式具有了singleTask模式的所有特性，还加强了一点，就是具有此启动模式的Act创建的时候只能单独地位于一个任务栈中。

例如：Act  A的启动模式是singleInstance，此时如果开始Act A，那么系统会创建一个新的任务栈，然后把Act A单独存放在这个新建的任务栈中，又因为singleInstance具有singleTask的所有特性，也就是说也有栈内复用的特性，如果当前的Act A不会销毁，那么以后再次启动Act A均不会创建新的Act。


例如：此时任务栈中有默认的Act A，此时开启一个singleInstance 模式的Act B，在Act B中开启一个非singleInstance的Act C，此时 Act C在Act A所在的栈中，在Act C中如果按返回键 此时返回看到的是Act A。
```

***
## Activity Flags

> Activity Flags有很多，这里主要分析一些比较常用的标记位。而标记位作用有很多。

> 比如：设定Activity 启动模式的 FLAG_ACTIVITY_NEW_TASK 和 FLAG_ACTIVITY_SINGLE_TOP等；影响Activity的运行状态的：FLAG_ACTIVITY_CLEAR_TOP 和 FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS等。

* **FLAG_ACTIVITY_NEW_TASK**

```
此标记位的作用是为Activity指定"singleTask"启动模式，作用效果和在XML中指定该启动模式相同。
```

* **FLAG_ACTIVITY_SINGLE_TOP**

```
此标记位的作用是为Activity指定"singleTop"启动模式，作用效果和在XML中指定该启动模式相同。
```

* **FLAG_ACTIVITY_CLEAR_TOP**

```
具有此标记位的Act，启动的时候，在同一任务栈中所有位于它上面的Act都要销毁。一般此标记位会和singleTask一起出现，在这种情况下，如果被启动的Act实例已经存在于当前任务栈中，那么系统会调用它的onNewIntent方法。如果此时配合默认的启动方式开启，那么此时会当任务栈中已经存在的实例连同它之上的Act都会销毁，清除出栈。系统会创建新的Act实例对象在栈顶。
```

* **FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS**

```
具有此标记位的Act，不会出现在历史Act列表中，当某些情况下我们不希望用户通过历史列表回到我们Act的时候，此标记会很好用。它等同于在XML文件中指定Act的属性"android:excludeFromRecents = "true" "。
```

## 具体情况：

* 项目中我们会做单点登录(多端登录)机制，举例说明：我们在BaseActivity中对后台返回code值做判断，假设404的时候判断为授权失效，此时跳转登录LoginActivity。

```
        if (code == 200) {
            return true;
        } else if (code == 404) {

            Intent intent = new Intent(context, LoginActivity.class);
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
            context.startActivity(intent);

            ToastUtils.initToast(msg);
            return false;
        } else {
            ToastUtils.initToast(msg);
            return false;

        }
```


```
上面代码中使用的是 addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)和addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP)，此时会重新开一个Task并写开启的Act具有singleTask属性。
```
* 项目中做推送的时候，一般会有通过推送回来的消息点击跳转页面，一般我们都在项目的入口Application中做推送的广播接收器，通过类型做具体跳转，此时我们开启Act的时候一般都是:

```
setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TOP);

```
此时是在同一个栈中开启一个Act，开启模式设置为singleTask。