## activity的启动流程
[启动流程总结文](https://cloud.tencent.com/developer/article/1917958)

以从Launcher 点击app图标启动为例：

#### Launcher进程用到的类：
- Launcher
- Activity
- Instrumentation
- ActivityTaskManager

服务进程用到的类：
- ActivityTaskManagerService
- ActivityStartController
- ActivityStarter
- ActivityStack

新app进程用到的类：
- ActivityThread
- ActivityManagerService
- ActivityTaskManagerService
- Instrumentation
- Activity
- Application
- ActivityTaskSupervisor
- ClientTransaction
- TransactionExecutor
- ClientTransactionHandler
- ActivityLifecycleItem（两个子类）

### 代码流程
#### launcher应用与AMS交互（请求创建与暂停自己）
用户点击图标，Launcher这个活动会调用startActivitySafely方法，其中会调用到Activity的
startActivity方法、startActivityForResult方法：
```
// Launcher.java
   public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
       ...
       //标记在新的栈启动
       intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
       ...
       startActivity(intent, optsBundle);
       ...
   }

 // Activity.java
   @Override
   public void startActivity(Intent intent, @Nullable Bundle options) {
       ...
       if (options != null) {
           //-1为requestCode表明不需要知道是否启动成功
           startActivityForResult(intent, -1, options);
       } else {
           startActivityForResult(intent, -1);
       }
   }

   public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
           @Nullable Bundle options) {
           ...
           Instrumentation.ActivityResult ar =
               mInstrumentation.execStartActivity(
                   this, mMainThread.getApplicationThread(), mToken,this,intent, requestCode, options);
           ...
    }
```
startActivityForResult内会使用Instrumentation.execStartActivity方法。每个activity内部都会持有Instrumentation对象。这个方法中传入了mMainThread.getApplicationThread()，它得到的是ActivityThread的内部类ApplicationThread的实例，来让服务得到Launcher的Application进程的binder对象，以便接下来让服务端使用它发起launcher活动的暂停。Instrumentation里调用ActivityTaskManager的service的startActivity方法，进入服务端。如下：
```
// Instrumentation.java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    ...
	int result = ActivityTaskManager.getService().startActivity(whoThread,who.getBasePackageName(),
        who.getAttributionTag(),intent,intent.resolveTypeIfNeeded(who.getContentResolver()),
        token,target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
    ...
}

 // ActivityTaskManager.java
public static IActivityTaskManager getService() {
       return IActivityTaskManagerSingleton.get();
}
private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
             }
};
```
进入ActivityTaskManagerService(在wm文件夹里)的startActivity方法,最后得到一个activityStarter(好像是通过工厂模式)：
```
// ActivityTaskManagerService.java
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }

    @Override
    public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }

    private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
	    ...
        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setUserId(userId)
                .execute();
    }

    ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }
// ActivityStarter.java
 int execute() {
   ...
   res = executeRequest(mRequest);
   ...
 }
//层层调用会到下面这个方法
ActivityStack.java
 private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    if (mResumedActivity != null) {
       pausing |= startPausingLocked(userLeaving, false , next);
    }
    ...
    mStackSupervisor.startSpecificActivity(next, true, false);
    ...
 }
```
调用到ActivityStack（对当前任务栈进行操作）的resumeTopActivityInnerLocked方法，其中调用startPausingLocked来通知launcher暂停。然后使用stackSupervisor.startSpecificActivity来在当前栈内启动目标活动：
```
// ActivityStackSupervisor.java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // 获取要启动的Activity进程信息
    final WindowProcessController wpc =
          mService.getProcessController(r.processName, r.info.applicationInfo.uid);
    boolean knownToBeDead = false;
    //如果进程存在且有进程中有线程存在 就是启动一个同应用的Activity（普通Activity就在此执行）
    if (wpc != null && wpc.hasThread()) {
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity " + r.intent.getComponent().flattenToShortString(), e);
        }
        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }
	//否则通过AMS向Zygote进程请求创建新的进程
    r.notifyUnknownVisibilityLaunchedForKeyguardTransition();
    final boolean isTop = andResume && r.isTopRunningActivity();
    mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
}
```
在stackSupervisor内会从mService得到该activity的wpc(WindowProcessController)，检查如果wpc已存在（则已存在进程）且其中有线程，则直接在该进程内启动，否则通过mService向Zygote进程请求创建新的进程。（mService是ActivityManagerService）。

#### app进程创建与appThread创建
 到此完成了Launcher与AMS通信，AMS与Zygote进程通信。Zygote启动新的进程时会标记ActivityThread.main函数。在Zygote创建好进程后通过反射调用main，之后便在新的App进程中进行；
```
ActivityThread.java
    // 内部类：
    final ApplicationThread mAppThread = new ApplicationThread();

    public static void main(String[] args) {
        ...
        Looper.prepareMainLooper();
	...
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);
	...
        Looper.loop();
	...
    }

    private void attach(boolean system, long startSeq) {
            final IActivityManager mgr = ActivityManager.getService();
            try {
                // activitythread的attach方法中，用AMS去处理appThread
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ...
    }
ActivityManagerService.java
    private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
            ...
            thread.bindApplication(processName, appInfo, providerList,
                        instr2.mClass,
                        profilerInfo, instr2.mArguments,
                        instr2.mWatcher,
                        instr2.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions,
                        app.mDisabledCompatChanges);
             ...
             didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
             ...
    }
```
main方法中创建了activity thread的looper对象；然后调用新创建的thread对象的attach方法，
attach方法中调用AMS的attachApplication（调到attachApplicationLocked）来将applicationThread绑定到AMS中。
attachApplication传入new的ApplicationThread对象（appThread的binder对象）。

进入到AMS中，AMS中的thread是appThread，调用它的bindApplication方法会发送一个H.BIND_APPLICATION消息，
对应handler的处理方法中调用handleBindApplication,最后使用Instrumentation调到Application的onCreate，
来准备好Application： 到此，启动新的App线程并且绑定了Application。
```
// Actvityhtread.java 内部类ApplicationThread中：
public final void bindApplication(...) {
    ... // H是ActivityThread的内部handler类
  	sendMessage(H.BIND_APPLICATION, data);
 }

public void handleMessage(Message msg) {
    switch (msg.what) {
       case BIND_APPLICATION:
       	AppBindData data = (AppBindData)msg.obj;
       	handleBindApplication(data);
       	Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
       	break;
        ...
      }
}

private void handleBindApplication(AppBindData data) {
	   ...
	  mInstrumentation.callApplicationOnCreate(app);
	   ...
}
```
#### activityThread处理Activity创建
appThread和application已经处理好了，该处理ActivityThread和create Activity了。
在AMS内还使用了mAtmInternal.attachApplication,这里会层层调用到ActivityStackSupervisor.realStartActivityLocked
方法，来处理Activity创建。mAtmInternal实际上是ActivityTaskManagerService的内部实现类LocalService(继承了ActivityTaskManagerInternal),所以去这里面看attachApplication方法，：
```
// ActivityTaskManagerService.java
@Override
public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
    synchronized (mGlobalLockWithoutBoost) {
        if (Trace.isTagEnabled(TRACE_TAG_WINDOW_MANAGER)) {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "attachApplication:" + wpc.mName);
        }
        try {
            return mRootWindowContainer.attachApplication(wpc);
            } finally {
                Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
            }
        }
}
```
看到里面调用了 mRootWindowContainer.attachApplication(wpc)。mRootWindowContainer是从WMS获取的，
rootWindow容器:
```
// RootWindowContainer.java :
attachApplication(WindowProcessController app) throws RemoteException {
    try {
       return mAttachApplicationHelper.process(app);
    } finally {
        mAttachApplicationHelper.reset();
    }
}
  // 内部实现类
private class AttachApplicationHelper implements Consumer<Task>, Predicate<ActivityRecord> {
    boolean process(WindowProcessController app) throws RemoteException {...}
    @Override
    public boolean test(ActivityRecord r) {
        ...
        mTaskSupervisor.realStartActivityLocked(r, mApp,
          mTop == r && r.getTask().canBeResumed(r) /* andResume */,true /* checkConfig */)
        ...
```
(此时在服务端)这一段调用到mTaskSupervisor.realStartActivityLocked，进入ActvityTaskSupervisor:
```
// com/android/server/wm/ActivityTaskSupervisor.java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
       boolean andResume, boolean checkConfig) throws RemoteException {
       ...
       final ClientTransaction clientTransaction = ClientTransaction.obtain(
                proc.getThread(), r.appToken);
         final DisplayContent dc = r.getDisplay().mDisplayContent;
         //这里生命周期添加的是LaunchActivityItem
         clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                 System.identityHashCode(r), r.info,
                 mergedConfiguration.getGlobalConfiguration(),
                 mergedConfiguration.getOverrideConfiguration(), r.compat,
                 r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                 r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                 dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                 r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));

         final ActivityLifecycleItem lifecycleItem;
         if (andResume) {  //直接继续或者创建后暂停，这里添加了ResumeActivityItem
             lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
         } else {
             lifecycleItem = PauseActivityItem.obtain();
         }
         clientTransaction.setLifecycleStateRequest(lifecycleItem);
         //执行clientTransaction
         mService.getLifecycleManager().scheduleTransaction(clientTransaction);
         ...
}
```
以上代码中，ClientTransaction管理了Activity的启动信息LaunchActivityItem继承ActivityLifecyleItem
而来，具体化了Activity的生命周期，并可由ClientLifecycleManager执行:从ActivityTaskManagerService
拿到ClientLifecycleManager，调用其scheduleTransaction执行这个clientTransaction
(mService是ActivityTaskManagerService)：
```
// （wm下面）ClientLifecycleManager.java
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    transaction.schedule();
    ...
}
// android/app/servertransaction/ClientTransaction.java
public void schedule() throws RemoteException {
    // mClient是IApplicationthread
    mClient.scheduleTransaction(this);
}
// android/app/ActivityThread.java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
class H extends Handler {
     ...
     public void handleMessage(Message msg) {
     ...
        case EXECUTE_TRANSACTION:
            final ClientTransaction transaction = (ClientTransaction) msg.obj;
            mTransactionExecutor.execute(transaction);
            if (isSystem()) {
                transaction.recycle();
            }
            break;
     ...
     }  
     ...
}
// android/app/servertransaction/TransactionExecutor.java
public void execute() {
    ...
    executeCallbacks(transaction);
    executeLifecycleState(transaction);
}
public void executeCallbacks(ClientTransaction transaction) {
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
    }
}
private void executeLifecycleState(ClientTransaction transaction) {
         final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
         ...
         // Execute the final transition with proper parameters.
         lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
         lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```
上面代码，在Lifecyclemanager里面调ClientTransaction的schedule，schedule里面调mClient的
scheduleTransaction方法，就又回到了ApplicationThread，给handler发送了H.EXECUTE_TRANSACTION，
在里面执行了这个transaction。执行由TransactionExecutor来处理，execute方法里面会调用
executeCallbacks和executeLifecycleState，一个用来执行之前在Transaction里设置的callback
里面的LaunchLifecycleItem，一个用来执行set的LifeCycleState。最终两个执行都调了lifecycleItem的execute。
```
// android/app/servertransaction/LaunchActivityItem.java
@Override
public void execute(ClientTransactionHandler client, IBinder token,
    PendingTransactionActions pendingActions) {
     ActivityClientRecord r = new ActivityClientRecord(...);
    client.handleLaunchActivity(r, pendingActions, mDeviceId, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
// android/app/servertransaction/ResumeActivityItem.java
@Override
public void execute(ClientTransactionHandler client, ActivityClientRecord r,
     PendingTransactionActions pendingActions) {
    client.handleResumeActivity(r, true /* finalStateRequest */, mIsForward,
}
```
这里面client就是Activitythread，activityThread在设置成员变量时设置TransactionExecutor，
初始化executor时传入了自己作为executor的ClientTransactionHandler，因此传给lifecycleItem的
client参数就是activityThread，所以这里又调回了activityThread的handleLaunchActivity和
handleResumeActivity方法：
```
// android/app/ActivityThread.java
public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        ...
        final Activity a = performLaunchActivity(r, customIntent);
        ...
        return a;
}
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ContextImpl appContext = createBaseContextForActivity(r);
        ...
        //反射创建Activity
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        ...
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ...
        activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);
        ...
        activity.setTheme(theme);
        ...
        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
}
// 同样的方法处理handleLaunchActivity：
public void handleResumeActivity(){
    performResumeActivity(r, finalStateRequest, reason)
};
 public boolean performResumeActivity(...){...}
```
performLaunchActivity方法中主要做了以下几件事：
1. 创建要启动activity的上下文环境
2. 通过Instrumentation的newActivity方法，以反射形式创建activity实例
3. 如果Application不存在的话会创建Application并调用Application的onCreate方法
4. 初始化Activity，创建Window对象（PhoneWindow）并实现Activity和Window相关联
5. 通过Instrumentation调用Activity的onCreate方法

自此调用了Activity的onCreate方法和onResume方法，一个新的活动就被启动起来了。
