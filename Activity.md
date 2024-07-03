# Activity
[Google官方文档](https://developer.android.com/guide/components/activities/intro-activities?index=..%2F..%2Fandroid-training&hl=zh-cn#0)

## 生命周期
[Google](https://developer.android.com/guide/components/activities/activity-lifecycle?hl=zh-cn)

## 任务栈
activity创建后会保存在任务栈结构中，因此其内的activity先进后出；

### 特性
1. 打开新的activity后，新的活动会被存在栈顶，且只有栈顶活动可以与用户交互；
2. 栈顶活动结束后，界面会退回到任务栈中的上一个活动；
3. 当一个任务栈中的活动全部结束（被退回），此时任务栈内无活动，将其销毁，并回退到最近一次访问的任务栈顶；
4. 系统桌面（Launcher程序）也是一个独立任务栈，当从桌面进入自己的任务栈，任务栈全部结束后会退回到Launcher界面；
5. 当按下Home按键/任务按键（多任务窗口）时，实际也是打开了Launcher程序，相当于进入任务栈一次；
6. 任务栈由操作系统管理，不属于某个应用或进程；
7. 以下行为会创建新的任务栈：
        - Launcher程序通过桌面图标启动一个新的活动；
        - Launcher通过通知栏启动了一个新的活动；
        - 当前活动启动了一个`SingleInstance`模式的活动；
        - 当前活动启动了一个制定了`taskAffinity`属性的活动；
        - 当前活动启动了一个`documentLaunchMode=“always”`的活动;
        - 当前活动启动活动时指定了`FLAG_ACTIVITY_NEW_TASK`选项；
8. 除了上述情况外，一个活动启动另一个活动，这两个活动都是处于同一任务栈中的（默认条件下），即便是多进程应用或调用其他应用中的活动，也会处于同一个任务栈中；
9. 即处在哪个任务栈和位于哪个进程线程无关，只与它的启动方式有关（被launcher启动或设置参数）；

### 获取应用任务栈信息
操作系统处于安全考虑，只允许我们获取和当前应用相关的任务栈
这里的相关，指的任务栈由当前应用创建，如果其它应用启动了我们的共享组件，只能在其它应用中获取该组件相关的任务栈信息

方法一：这个API只能获取到最近任务列表中的任务栈
后台任务栈必须开启了android:documentLaunchMode=“always”，出现在最近任务列表中，才能获取到；
```
	//获取任务栈列表
	ActivityManager activityManager = getSystemService(ActivityManager.class);
	List<ActivityManager.AppTask> tasks = activityManager.getAppTasks();

	//遍历任务栈
	for (ActivityManager.AppTask task : tasks) {
	    ActivityManager.RecentTaskInfo info = task.getTaskInfo();

		int taskId = info.id; //任务栈ID

	    Intent launchIntent = info.baseIntent; //创建该任务栈的Intent
	    String action = launchIntent.getAction(); //Intent的action
	    Set<String> categories = launchIntent.getCategories(); //Intent的category
	    ComponentName component = launchIntent.getComponent(); //Intent要启动的组件名，即Activity的包名和类名
	    Bundle extras = launchIntent.getExtras(); //Intent携带的额外参数

	    int activityCount = info.numActivities; //栈内Activity数量
	    String topActivity = info.topActivity.getClassName(); //栈顶Activity
	    String baseActivity = info.baseActivity.getClassName(); //栈底Activity，如果所有Activity都不销毁，baseActivity就是originActivity

	    boolean isRunning = info.isRunning; //任务栈是否在运行，Activity有可能处于挂起状态，等待restore，该API安卓10开始才可用

	    task.moveToFront(); //调度任务栈至前台显示
	    task.setExcludeFromRecents(true); //不显示在最近任务列表中
	    task.finishAndRemoveTask(); //销毁任务栈内全部Activity
	    task.startActivity(context, intent, options); //在指定任务栈中启动一个新的Activity，并将当前任务栈调至前台
	}
```
方法二：能获取应用全部的任务栈，但接口没有上面灵活；
```
		//获取任务栈列表
	   ActivityManager manager = getSystemService(ActivityManager.class);
	   List<ActivityManager.RunningTaskInfo> infos = manager.getRunningTasks(100);

	   //遍历任务栈
	   for (ActivityManager.RunningTaskInfo info : infos) {
	       int taskId = info.id;
	       int activityCount = info.numActivities;
	       ComponentName baseActivity = info.baseActivity;
	       ComponentName topActivity = info.topActivity;
	       manager.moveTaskToFront(taskId, 0);
	       break;
	   }
```

## 四种启动模式
1. Standard模式：默认模式，启动一个活动就将其压入栈顶；
2. SingleTop模式：单顶模式，复用栈顶，栈顶没有再在栈顶创建并使用；
3. SingleTask模式：单任务模式，栈内复用，只要该任务栈内存在活动就会复用，当前任务跳转到目标任务时，即从当前栈顶移动到目标活动所在位置时，删除其之间的所有活动，所以最后目标活动会处在栈顶；
4. SingleInsance模式：纯单例模式，该模式每次都会创建新任务栈，且栈内只有这一个活动；

### 几种特殊启动模式的应用场景

 SingleTop模式
比如我们使用一款视频APP观看短视频，视频播放页面同时还推荐了类似视频
当我们点击了推荐视频，事件顺序大概是：打开视频播放页面 - 点击推荐视频 - 复用当前页面播放推荐视频，同时刷新推荐列表
即栈顶对象可复用，没必要新建一个实例

 SingleTask模式
比如我们使用一款购物APP购买商品，Activity打开顺序大概是：商品页面 - 下单页面 - 支付页面 - 交易完成界面
显然，我们完成时，需要返回商品页面，没必要返回支付页面，因为订单已经结束，因此我们需要在回到商品页面时销毁下单和支付页面
即Activity顶部的任务已过期，没必要再保留

 SingleInstance模式
比如我们使用系统自带的相机应用，由于性能和硬件占用的原因，肯定是不允许同时打开两份的
当我们进去相机开始录像，再去其它应用里面逐个逛一遍，然后回到桌面再点击相机图标，肯定是会回到之前的实例里继续录制，而不是新建一个新的实例，也不能销毁改Activity顶部的其它Activity
即客观条件只允许单例，但栈结构并不允许复用栈中间的对象，只能复用栈顶对象（ SingleTop），或者先弹出复用对象顶部的其它对象，让目标对象浮到栈顶再复用（SingleTask），所以SingleInstance模式只能自己独占一个任务栈

### SingleInstance模式弊端

SingleInstance模式将应用变成了多任务栈应用，如果两个任务栈之间启动过其它应用，将会导致应用回退混乱

由于安卓在任务栈栈销毁时，会回退到最近一次启动过的其它任务栈，这样如果同一个应用的栈A和栈B中间启动过其它应用的任务栈的话，当栈B按返回结束后，就不会跳到栈A，而是跳到其它应用

由于安卓桌面本身也是一个应用，也占用了一个任务栈，通过通知栏或后台工作，其它应用也可能中途启动，这种情况是经常发生的

一个常见的情景就是：栈A启动 - 栈B启动 - 按下Home键回到桌面 - Launcher栈启动 - 回到栈B - 栈B销毁 - 回到Launcher栈（即回到桌面）

### 解决SingleInstance模式下，按下返回键不返回上个Activity，而是返回桌面的问题

通过以上的原理阐述，我们已经能够很容易的解释这种现象

假如我们的应用是个多任务栈应用，包含了任务栈A，任务栈B，桌面程序使用的是Launcher任务栈
我们在任务栈A中启动了任务栈B，然后按下了Home键或任务键，又回到了任务栈B，然后按下了返回键
由于按下Home键或任务键，就等于启动了Launcher任务栈，那么任务栈B之前最近一次启动的任务栈就是Launcher，任务栈B销毁后就得返回Launcher

解决方案很简单，由于我们可以获取到当前应用的任务栈列表，我们将任务栈列表中的其它任务栈调度至前台即可

但是这个方法仅适合双任务栈的情景，N个任务栈的情景则需要大家根据业务去变通处理
好在任务栈信息是可获取的，大家完全可以在Activity创建和销毁时，记录任务栈调度过程，写一个自己的任务栈管理规则，这样再复杂的情景都不是问题

上面提到了两种获取任务栈的方式，这里我们使用第二种方式来获取任务栈信息
```

    @Override
    public void onBackPressed() {
		//获取任务栈列表
	   ActivityManager manager = getSystemService(ActivityManager.class);
	   List<ActivityManager.RunningTaskInfo> infos = manager.getRunningTasks(100);

	   //遍历任务栈
	   for (ActivityManager.RunningTaskInfo info : infos) {
	       int taskId = info.id;
	       ComponentName baseActivity = info.baseActivity;
	       ComponentName topActivity = info.topActivity;
	       int activityNum = info.numActivities;
	       //跳过当前任务栈
	       if (taskId == getTaskId()) continue;
	       //跳过外部任务栈，一般为Launcher程序的任务栈
	       if (!Texts.equal(topActivity.getPackageName(), getPackageName())) continue;
	       //将任务栈调度至前台
	       //使用ActivityManager.MOVE_TASK_WITH_HOME作为flag，可以将任务调至栈顶时，同时将Home程序的任务栈调至该任务之下
	       //这样当用户结束栈顶任务时，就会回退到Launcher任务栈，即回到桌面，这个功能可用于特殊场景
	       manager.moveTaskToFront(taskId, 0);
	       break;
	   }

	   //将当前任务栈移至后台
	   moveTaskToBack(true);
	   //销毁当前任务栈
	   finishAndRemoveTask();
    }


```
### 任务栈高级选项
除了以上提到的基本功能之外，安卓还提供一些高级选项，可以在Manifest中进行配置
```
	<activity
		android:name=".other.ThirdActivity"
	    android:taskAffinity=":xxx_task"
	    android:documentLaunchMode="always"
	/>
```
taskAffinity：任务栈关联，新开一个任务栈，相同taskAffinity的Activity会放到同一个任务栈，该属性必须以冒号和英文开头。启动Activity时，只有携带了Intent.FLAG_ACTIVITY_NEW_TASK标志，taskAffinity属性才会生效。设置了taskAffinity的任务栈会在最近任务列表中通过一个单独的缩略图来显示
 taskAffinity属性的一个用途是，限制SingleTask销毁顶部的所有Activity，比如Activity的打开顺序是：X - A - B - C - D - E - X，如果我们想X第二次启动时，销毁DE，而保留ABC，我们可以将XDE放在同一个栈中，设置相同的taskAffinity，并将X设为SingleTask模式
 documentLaunchMode：文档模式，在编程中，习惯把顶级应用下的独立的字窗口叫做文档。文档模式会给Activity新开一个任务栈，并且在最近任务列表中通过一个单独的缩略图来显示。同一个Activity如果有多个实例，每个实例都是一个独立的任务栈。设置了documentLaunchMode后，taskAffinity属性无效
 allowTaskReparenting：任务重新绑定任务栈，一般用于共享Activity。比如应用A打开了应用B的共享组件ShareActivity，由于是从应用A打开的，ShareActivity会保存到应用A的任务栈中。但如果我们此时启动应用B，应用B将不会从LoginActivity启动，而是直接从ShareActivity启动，并将ShareActivity从应用A的任务栈迁移到应用B的任务栈中
 alwaysRetainTaskState：在Activity被后台清理时，是否保留任务栈状态。如果设置为了false，任务栈的状态有可能不被保存，下次打开应用时，任务栈将从栈底的RootActivity重新启动。如果设置为true，应用则会重现创建所有的Activity，恢复之前的任务栈状态，并回到之前栈顶的TopActivity
 clearTaskOnLaunch：点击桌面图标，重现启动应用时，是否清理之前的任务栈状态。如果设置为true，则会清理之前的任务栈状态，等于是完全重新启动。如果设置为false，则会回到上次离开应用时所在的Activity，一般情况下都为false
 autoRemoveFromRecents：当任务栈为空时，是否自动从最近任务列表中清除缩略图

### 解决SingleInstance模式下，点击桌面图标没有回到上个Activity，而是自动重启的问题

 这个问题网上很多都是通过判断isTaskRoot的方法来解决的，但是没有讲解过原理
 实际上，这并不是一个通用的解决方法，仅适合某些APP的任务栈管理情景，想要根本解决这个问题，还是要弄懂原因

 假如我们应用的启动顺序是SplashActivity - LoginActivity - MainActivity
 其中MainActivity是SingleInstance模式，独占一个任务栈，其它两个Activity是普通模式
 SplashActivity是Application启动后的首个Activity，我们把它叫做入口Activity，把它所在的任务栈叫做Standard任务栈

 点击桌面图标后，Launcher做的工作是：检查Standard任务栈是否存在，存在就打开Standard任务栈的栈顶Activity，不存在就新建一个Standard任务栈，并在该任务栈中启动入口Activity

 假如我们从MainActivity回到桌面，再点击应用图标，这时应用怎么做，实际上取决于SplashActivity和LoginActivity有没有被销毁。但不管怎么样，肯定不会回到MainActivity，因此Launcher要启动的是Standard任务栈

 如果LoginActivity没有被销毁，那么LoginActivity就处于Standard任务栈的栈顶，返回到LoginActivity
 如果LoginActivity和SplashActivity在跳转到其它Activity时都销毁了自己，Standard任务栈就是空的，也会被销毁，从Launcher再点击图标时，就会重建Standard任务栈和LoginActivity实例

 值得一提的是，Standard任务栈和LoginActivity重建，并不代表整个应用重建了。进程还是之前的那个进程，MainActivity实例仍然是存活的。Launcher的目的并不是为了重启应用，仅仅是因为工作机制如此

 原理弄清楚了，解决问题其实就简单了。我们只需解决两个问题，一个是怎么判断应用是首次启动还是第二次启动，另一个是怎么回到MainActivity

 判断应用是否首次启动其实很简单，我们记录下Application的启动时间，对比下时间间隔就知道是否首次启动了，不是首次启动就销毁SplashActivity和LoginActivity

 如果只有一个SingleInstance模式的Activity的话，直接启动MainActivity就行了。如果有多个SingleInstance模式的Activity的话，我们可以在onCreate方法中记录下每个Activity的启动顺序，找到最后一次启动的Activity的类名，启动它就行了

 对于一般应用而言，SingleInstance模式和多进程的Activity并不会特别多。弄清了原理，我们根据情况变通就行了
 ```
 	long interval = Times.millisOfNow() - CommonApplication.applicationStartTime;
 	if (interval > 1000) {
 	    finish();
 	    start(MainActivity.class);
 	    return;
 	}
 ```
### 应用退出时从最近任务列表中清除缩略图

通过上面提到的autoRemoveFromRecents属性，就可以很简单的达到这个效果
但是我们需要为每个任务栈都设置一次，我们也可以通过代码，在基类Activity中统一清除
```
	@Override
	protected void onDestroy() {
	    super.onDestroy();

	    //获取任务栈列表
	    ActivityManager manager = getSystemService(ActivityManager.class);
	    List<ActivityManager.AppTask> tasks = manager.getAppTasks();

	    //遍历任务栈，找到当前任务栈
	    //如果任务栈没有其它Activity，则从最近任务列表中移除
	    for (ActivityManager.AppTask task : tasks) {
	        ActivityManager.RecentTaskInfo info = task.getTaskInfo();
	        if (info.numActivities == 0)
	            task.finishAndRemoveTask();
	    }
	}
```

## 重启时信息传递savedInstanceBundle
将本次activity关闭后的一些必要数据保存，下次create可以直接显示出来。
```
TextView textView;

// Some transient state for the activity instance.
String gameState;

@Override
public void onCreate(Bundle savedInstanceState) {
    // Call the superclass onCreate to complete the creation of
    // the activity, like the view hierarchy.
    super.onCreate(savedInstanceState);

    // Recover the instance state.
    if (savedInstanceState != null) {
        gameState = savedInstanceState.getString(GAME_STATE_KEY);
    }

    // Set the user interface layout for this activity.
    // The layout is defined in the project res/layout/main_activity.xml file.
    setContentView(R.layout.main_activity);

    // Initialize member TextView so it is available later.
    textView = (TextView) findViewById(R.id.text_view);
}

// This callback is called only when there is a saved instance previously saved using
// onSaveInstanceState(). Some state is restored in onCreate(). Other state can optionally
// be restored here, possibly usable after onStart() has completed.
// The savedInstanceState Bundle is same as the one used in onCreate().
@Override
public void onRestoreInstanceState(Bundle savedInstanceState) {
    textView.setText(savedInstanceState.getString(TEXT_VIEW_KEY));
}

// Invoked when the activity might be temporarily destroyed; save the instance state here.
@Override
public void onSaveInstanceState(Bundle outState) {
    outState.putString(GAME_STATE_KEY, gameState);
    outState.putString(TEXT_VIEW_KEY, textView.getText());

    // Call superclass to save any view hierarchy.
    super.onSaveInstanceState(outState);
}
```
`onSaveInstanceState`的几个注意事项：
1.在一个activity被销毁前，不一定会调用onSaveInstanceState()这个方法，因为不是所有情况都需要去存储activity的状态（例如当用户按回退键退出你的activity的时候，因为用户指定关掉这个activity）。
2.如果这个方法被调用，它一定会在 onStop()方法之前，可能会在onPause()方法之前。
3.布局中的每一个View默认实现了onSaveInstanceState()方法，这样的话，这个UI的任何改变都会自动的存储和在activity重新创建的时候自动的恢复。但是这种情况只有在你为这个UI提供了唯一的ID之后才起作用，如果没有提供ID，将不会存储它的状态。
4.由于默认的onSaveInstanceState()方法的实现帮助UI存储它的状态，所以如果你需要覆盖这个方法去存储额外的状态信息时，你应该在执行任何代码之前都调用父类的onSaveInstanceState()方法（super.onSaveInstanceState()）。
5.由于onSaveInstanceState()方法调用的不确定性，你应该只使用这个方法去记录activity的瞬间状态（UI的状态）。不应该用这个方法去存储持久化数据。当用户离开这个activity的时候应该在onPause()方法中存储持久化数据（例如应该被存储到数据库中的数据）。

## 其他
AppCompatActivity  ---  向下兼容带有新特性的activity基类。extends FragmentActivity,多了一个AppCompatDelegate

`import androidx.core.app.ActivityCompat;`activity辅助类，权限处理等。

## 生命周期感知型组件
[Google](https://developer.android.com/topic/libraries/architecture/lifecycle?hl=zh-cn)


## 参考链接
[任务栈与启动模式](https://blog.csdn.net/u013718730/article/details/108362585)
[四种启动模式](https://blog.csdn.net/qq_37980878/article/details/107533764)
