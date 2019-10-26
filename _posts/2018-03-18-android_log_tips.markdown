---
layout:     post
title:      "Log 技巧"
subtitle:   "Android Log 技巧"
date:       2018-03-18 00:10:40
author:     "Mathew"
catalog: true
header-img: "img/post-bg-2017.jpg"
tags:
    - Android
    - Log 技巧
---
# Log技巧

## 常用搜索log的技巧

### 通知的技巧
1. 通知的显示
弹出通知必定会走的log
```java
//TAG 是NotificationService
Slog.v(TAG, "enqueueNotificationInternal: pkg=" + pkg + " id=" + id
        + " notification=" + notification);
```
典型的通知的log
```java
01-01 17:55:06.411   955  2178 V NotificationService: enqueueNotificationInternal: pkg=com.android.deskclock id=1 notification=Notification(pri=2 contentView=null vibrate=null sound=null defaults=0x0 flags=0x102 color=0x00000000 category=alarm actions=1 vis=PUBLIC)
```
典型通知的log的参数说明
```java
pkg: //包名
id: //通知的id
pri: //通知的优先级
contentView: //通知的内容视图
virabte: //是否震动
sound: //声音的Uri
defaults: //DEFAULT_SOUND DEFAULT_VIBRATE DEFAULT_LIGHTS DEFAULT_ALL 前三个的与或非, 或者是最后一个值, 表示通知是否有声音, 震动, 通知灯, 或者三者都有 
flags: //没明白这个
color: //通知灯的颜色, 有些机器可以通过三基色形成各种各样的颜色
category: //通知的类别CATEGORY_CALL, CATEGORY_MESSAGE, CATEGORY_EMAIL 等类别
actions: //通知的含有的action
vis: //通知的可见属性, 只会取VISIBILITY_PUBLIC, VISIBILITY_PRIVATE, VISIBILITY_SECRET具体什么意思, 查看源码
```

Heads-up Notification 通知一定会走的log, 以及在通知栏会显示
```java
//TAG 是NotificationService
if (DEBUG) Log.d(TAG, "addNotification key=" + notification.getKey());
```
2. 通知声音一定会走的log

```
//TAG 是NotificationService
if (DBG) Slog.v(TAG, "Playing sound " + soundUri + " with attributes " + audioAttributes);

```

3. 通知SystemUI里有的log
```java
//TAG StatusBar
Log.d(TAG, "onNotificationPosted: " + sbn);
```

4. 一定会走的log
```java
//TAG NotificationService
if (DBG) Slog.d(TAG, "EnqueueNotificationRunnable.run for: " + n.getKey());
```

5. 移除通知的log, SystemUI一定会走的log
```java
//TAG StatusBar
Log.d(TAG, "onNotificationRemoved: " + sbn);
```

### 弹出Toast的技巧
弹出的Toast必定会走的log
```java
//TAG 是NotificationService
Slog.i(TAG, "enqueueToast pkg=" + pkg + " callback=" + callback
        + " duration=" + duration);

```

### 取消Toast一定会走的log

```java
Slog.i(TAG, "cancelToast pkg=" + pkg + " callback=" + callback);
```

### App启动Log

```java
if (SHOW_ACTIVITY_START_TIME) {
    Trace.asyncTraceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER, "launching: " + packageName, 0);
    EventLog.writeEvent(EventLogTags.AM_ACTIVITY_LAUNCH_TIME,
            userId, System.identityHashCode(this), shortComponentName,
            thisTime, totalTime);
    StringBuilder sb = service.mStringBuilder;
    sb.setLength(0);
    sb.append("Displayed ");
    sb.append(shortComponentName);
    sb.append(": ");
    TimeUtils.formatDuration(thisTime, sb);
    if (thisTime != totalTime) {
        sb.append(" (total ");
        TimeUtils.formatDuration(totalTime, sb);
        sb.append(")");
    }
    Log.i(TAG, sb.toString());

    /// M: Add BootEvent for profiling
    BootEvent.addBootEvent("AP_Launch: " + shortComponentName +
            " " + thisTime + "ms");
}
```

### fork新进程的log

#### 新启一个Android进程一定会走的log

```cpp
ALOGI("%s: Begin to fork a new process",__FUNCTION__);
ALOGI("%s: fork is finished, pid of child process is %d ", __FUNCTION__, pid);
```

```cpp
Zygote  : ForkAndSpecializeCommon: fork is finished, pid of child process is 1694 
```


### 按键的log

KeyEvent




###  WIFI链接的log
大概是这个

 
### 亮灭屏幕一定会走的log
```java
 if (DEBUG) Log.v(TAG, "onReceive: " + intent);
 PhoneStatusBar: onReceive: Intent { act=android.intent.action.SCREEN_OFF flg=0x50000010 }
 PhoneStatusBar: onReceive: Intent { act=android.intent.action.SCREEN_ON flg=0x50000010 }
```

### 闹钟一定会走的log





### 日历发通知一定会走的log

```java
LogUtil.d(TAG, "postNotification(), notify notification:" + notification.mNotification + ",quietUpdate:"
        + quietUpdate + ",newAlert:" + info.newAlert + ",ringtone:" + ringtone);
```


```java
if (DEBUG) {
    Log.d(TAG, "Posting individual alarm notification, eventId:" + info.eventId
            + ", notificationId:" + notificationId
            + (TextUtils.isEmpty(ringtone) ? ", quiet" : ", LOUD")
            + (highPriority ? ", high-priority" : ""));
}
```

###  未知,  可能重要的Log	
```java
Log.v("PhoneWindow", "DecorView setVisiblity: visibility = " + visibility
    + ", Parent = " + getParent() + ", this = " + this);
```


### 屏幕焦点切换的log
猜测的, 并没有验证过, 没有研究过
```java
if (DEBUG_FOCUS_LIGHT || localLOGV) Slog.v(TAG_WM, "Changing focus from " +
       mCurrentFocus + " to " + newFocus + " Callers=" + Debug.getCallers(4));
```

### 触摸事件

```java
if (DEBUG) {
    Slog.d(TAG, "onUserActivity: event=" + event + ", uid=" + uid);
}


```
```java
event = 2 是 触摸事件的down事件

```

触摸事件只需要down一次
触摸事件down, up各一次的log, 这个log在InputReader.cpp文件里面

触摸事件up
```cpp
{
    ScopedTrace _l(ATRACE_TAG_INPUT|ATRACE_TAG_PERF, "AppLaunch_dispatchPtr:Up");
    ALOGD("AP_PROF:AppLaunch_dispatchPtr:Up:%lld, ID:%d, Index:%d", when/1000000, upId,
             mLastCookedState.cookedPointerData.idToIndex);
    ALOGD_READER("dispatchMotion POINTER UP now(ns): %lld",now);
    dispatchMotion(when, policyFlags, mSource,
        AMOTION_EVENT_ACTION_POINTER_UP, 0, 0, metaState, buttonState, 0,
        mLastCookedState.cookedPointerData.pointerProperties,
        mLastCookedState.cookedPointerData.pointerCoords,
        mLastCookedState.cookedPointerData.idToIndex,
        dispatchedIdBits, upId, mOrientedXPrecision, mOrientedYPrecision, mDownTime);
    dispatchedIdBits.clearBit(upId);
}

```
触摸事件down
```cpp
if (dispatchedIdBits.count() == 1) {
    // First pointer is going down.  Set down time.
    mDownTime = when;
    {
        ScopedTrace _l(ATRACE_TAG_INPUT, "AppLaunch_dispatchPtr:Down");
        ALOGD("AP_PROF:AppLaunch_dispatchPtr:Down:%lld, ID:%d, Index:%d", mDownTime/1000000, downId,
                     mCurrentCookedState.cookedPointerData.idToIndex);
	}
}

```
不大明白里面的参数什么意思, 
```java
12-21 14:45:06.067556   936  1037 D InputReader: AP_PROF:AppLaunch_dispatchPtr:Down:14956077, ID:0, Index:-1551259760
12-21 14:45:06.113281   936  1037 D InputReader: AP_PROF:AppLaunch_dispatchPtr:Up:14956123, ID:0, Index:-1551256368
```


```java
12-21 14:45:48.564368   936   936 I Tethering: StateReceiver onReceive action:android.intent.action.CONFIGURATION_CHANGED
```


### 非常重要, 但不知道表示什么意思的log
没研究过这个BufferQueueProducer, 不大明白
```cpp
#define BQP_LOGI(x, ...) LOG_PRI(ANDROID_LOG_INFO, "BufferQueueProducer", "[%s](this:%p,id:%d,api:%d,p:%d,c:%d) " x, mConsumerName.string(), mBq.unsafe_get(), mId, mConnectedApi, mProducerPid, mConsumerPid, ##__VA_ARGS__)
```


### 屏幕被唤醒的原因

```java
if (DEBUG) {
    Slog.d(TAG, "onWakeUp: event=" + reason + ", reasonUid=" + reasonUid
            + " opPackageName=" + opPackageName + " opUid=" + opUid);
}
```


### 触摸事件被消耗的log

一直想找到点击, 触摸等事件是被那个控件消费的, 但一直没找到



### RingtonePicker

```java
MtkLog.d(TAG, "onCreate: mHasDefaultItem = " + mHasDefaultItem
        + ", mUriForDefaultItem = " + mUriForDefaultItem
        + ", mHasSilentItem = " + mHasSilentItem
        + ", mHasMoreRingtonesItem = " + mHasMoreRingtonesItem
        + ", mType = " + mType
        + ", mExistingUri = " + mExistingUri);
```

RingtoneManager的getcursor的log
```java
Log.v(TAG, "mCursor.hashCode " + mCursor.hashCode());
Log.v(TAG, "getCursor with new cursor = " + mCursor);
```

### 启动四大组件的log

#### 新启动一个activity的时候出现的log: 

```java
if (err == ActivityManager.START_SUCCESS) {
    Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
            + "} from uid " + callingUid
            + " on display " + (container == null ? (mSupervisor.mFocusedStack == null ?
            Display.DEFAULT_DISPLAY : mSupervisor.mFocusedStack.mDisplayId) :
            (container.mActivityDisplay == null ? Display.DEFAULT_DISPLAY :
                    container.mActivityDisplay.mDisplayId)));
}
```
#### 启动一个Service一定会走的log

```java
// TAG ActivityThread
Slog.v(TAG, "SVC-Creating service " + data);
```

#### Serivce启动, Eng版的才会出现的log

```java
if (!IS_USER_BUILD || DEBUG_SERVICE) {
    Slog.d(TAG, "SVC-Executing service done: " + token
        + ", type=" + type
        + ", startId=" + startId
        + ", res=" + res);
}
```

#### Service启动到onStartCommand

```java
Slog.d(TAG, "SVC-Calling onStartCommand: " + s
        + ", flags=" + data.flags
        + ", startId=" + data.startId);

```

#### Service解绑, eng版才有的log
```java
if (!ActivityManagerService.IS_USER_BUILD || DEBUG_SERVICE) {
    Slog.d(TAG_SERVICE, "SVC-Unbinding service: " + r.binding.service
        + ", app=" + r.binding.service.app);
}

```

#### Service启动一定会走的log


```java
//TAG ActivityManager
if (!ActivityManagerService.IS_USER_BUILD || DEBUG_SERVICE) {
    Slog.v(TAG_SERVICE, ">>> EXECUTING "
        + why + " of " + r + " in app " + r.app);
} else if (DEBUG_SERVICE_EXECUTING) {
    Slog.v(TAG_SERVICE_EXECUTING, ">>> EXECUTING "
        + why + " of " + r.shortName);
}

```

```java
Slog.d(TAG, "SVC-Calling onStartCommand: " + s
        + ", flags=" + data.flags
        + ", startId=" + data.startId);
/// @}
```


####  发送广播的log

##### 发送广播一定会走的log

```java
Slog.v(TAG_BROADCAST,
        (sticky ? "Broadcast sticky: ": "Broadcast: ") + intent
        + " ordered=" + ordered + " userid=" + userId
        + " callerApp=" + callerApp);
```
##### 广播接收一定会走的log
```java
Slog.d(TAG, "BDC-Calling onReceive"
    + ": intent=" +  data.intent
    + ", receiver=" + receiver);

```

```java
// Tag是 BroadcastQueue
if (!ActivityManagerService.IS_USER_BUILD || DEBUG_BROADCAST) {
    Slog.d(TAG_BROADCAST, "BDC-Delivering broadcast: " + r.intent
        + ", queue=" + mQueueName
        + ", ordered=" + r.ordered
        + ", app=" + app
        + ", receiver=" + r.receiver);
}

```

##### 如果广播因为权限的原因传播失败
```java
String msg = "Permission Denial: not allowed to send broadcast "
        + action + " to "
        + intent.getComponent().getPackageName() + " from "
        + callerPackage;
Slog.w(TAG, msg);  //不一定是这个log, 但是tag和"Permission Denial:"一定有

```

#####	屏幕旋转的时候发送的广播
这个是在Slog出现的
```
12-21 14:45:48.447876   936   953 V ActivityManager: Broadcast: Intent { act=android.intent.action.CONFIGURATION_CHANGED flg=0x70000010 } ordered=false userid=-1 callerApp=null
12-21 14:45:48.571408   936   936 I AppWidgetServiceImpl: Received broadcast: android.intent.action.CONFIGURATION_CHANGED on user -10000
12-21 14:45:48.571462   936   936 I AppWidgetServiceImpl: onConfigurationChanged()

```



### 重启一定会走的log

```java
//TAG 是ShutdownThread
Log.d(TAG, "reboot");
```

```java
//TAG 是ShutdownThread
Log.d(TAG, "Notifying thread to start shutdown longPressBehavior=" + longPressBehavior);	
```

```java
Log.d(TAG, "PowerOff dialog doesn't exist. Create it first");
```

```
//TAG 是ShutdownThread
if (mSpew) {
    StackTraceElement[] stack = new Throwable().getStackTrace();
    for (StackTraceElement element : stack)
    {
        Log.d(TAG, "     |----" + element.toString());
    }
}
```
### 关机动画一定会走的log

```
//TAG 是ShutdownThread
Log.d(TAG, "mIBootAnim.isCustBootAnim() is true");
```


```
//TAG 是ShutdownThread
Log.i(TAG, "set service.shutanim.running to 0");
```


### MediaRecorder 必走的log

#### MediaRecorder创建的时候必走的log

```java
ALOGV("setup");
 
```

#### 

```java
ALOGV("setAudioSource(%d)", as);
```


### Telephony的log

### 挂断电话的log
```java
D AT      : AT< +ECPI: 1,130,0,0,0,0,"10086",129,""
3GPP specific cause values for call control  电话挂断的原因

```


#### 接电话一定会走的log

M平台
```java
Log.d(this, "Will show \"answer\" action in the incoming call Notification");
```




#### 拒接电话一定会走的log
M平台
```java
Log.d(this, "Will show \"dismiss\" action in the incoming call Notification");
```



### 触摸事件一定会走的log

```java
if (event.getAction() == KeyEvent.ACTION_DOWN) {
    Log.i(VIEW_LOG_TAG, "Key down dispatch to " + this + ", event = " + event);
} else if (event.getAction() == KeyEvent.ACTION_UP) {
    Log.i(VIEW_LOG_TAG, "Key up dispatch to " + this + ", event = " + event);
}
```




### ListView 的log

```java
Log.d(TAG, "item no. = " + mItemCount + ", adapter no. = " + mAdapter.getCount());
```



### Exchange 出现发送失败就会出现的log

```java
05-04 05:19:06.336 W/Exchange( 3728): General failure sending message: 4
05-04 05:19:06.338 E/Exchange( 3728): Generic error for operation SendMail: status 200, result -102
05-04 05:19:06.338 W/Exchange( 3728): Aborting outbox sync for error -99

```


### Eng版的就会出来的log
```java
if (false == IS_USER_BUILD) {
    Log.d(TAG, "interceptKeyTq keycode=" + keyCode
        + " interactive=" + interactive + " keyguardActive=" + keyguardActive
        + " policyFlags=" + Integer.toHexString(policyFlags)
        + " down =" + down + " canceled = " + canceled
        + " isWakeKey=" + isWakeKey
        + " mVolumeDownKeyTriggered =" + mScreenshotChordVolumeDownKeyTriggered
        + " mVolumeUpKeyTriggered =" + mScreenshotChordVolumeUpKeyTriggered
        + " result = " + result
        + " useHapticFeedback = " + useHapticFeedback
        + " isInjected = " + isInjected);
}
```


### 系统Service启动的log
```java
Slog.i(TAG, "Starting " + name);
```


#### 




eng版的才会出现的log, 处理各种事件

```java
if (DEBUG_INPUT || DEBUG_KEY || DEBUG_MOTION || DEBUG_DEFAULT) {
    Log.v(mTag, "enqueueInputEvent: event = " + event + ",processImmediately = "
            + processImmediately + ",mProcessInputEventsScheduled = "
            + mProcessInputEventsScheduled + ", this = " + this);
}
```
典型的log

```
01-01 05:15:10.047 V/ViewRootImpl[Launcher]( 1598): enqueueInputEvent: event = KeyEvent { action=ACTION_DOWN, keyCode=KEYCODE_VOLUME_UP, scanCode=115, metaState=0, flags=0x8, repeatCount=0, eventTime=15110785, downTime=15110785, deviceId=7, source=0x101 },processImmediately = true,mProcessInputEventsScheduled = false, this = ViewRoot{eb23f95 com.android.launcher3/com.android.launcher3.Launcher,ident = 0}
//[]里面的表示当前是哪个模块, 或者app, 相应的参数稍后再详细说明
```



```java
Log.d("WindowClient", "Add to mViews: " + view + ", this = " + this);
```


### Cell Broadcast 广播
在CellBroadcastHandler.java类里面
```java
//紧急的小区广播
log("Dispatching emergency SMS CB, SmsCbMessage is: " + message);

```

```java
// 普通的小区广播
log("Dispatching SMS CB, SmsCbMessage is: " + message);
```
CellBroadcastHandler的父类是WakeLockStateMachine. 默认在Radio log里面
```java
protected void log(String s) {
    Rlog.d(getName(), s);
}
```
### 命令行打开log
命令行动态打开

简写
命令含义
命令行
```
x	打开所有的开关	adb shell dumpsys activity log x on
a	activity相关	adb shell dumpsys activity log a on
da	查看OOM_ADJ等，一般用于Debug Memory问题时用	adb shell dumpsys activity log da on
br	Broadcast相关	adb shell dumpsys activity log br on
s	Service相关	adb shell dumpsys activity log s on
cp	ContentProvider相关	adb shell dumpsys activity log cp on
p	Permission相关	adb shell dumpsys activity log p on
lp	打开某个进程的looper	adb shell dumpsys activity log lp 进程名
anr	ANR相关	adb shell dumpsys activity log anr 2
 修改代码的方式打开(一般用于分析开机慢或进入launcher慢等问题)

/frameworks/base/services/core/java/com/android/server/am/ActivityManagerDebugConfig.java
```
打开所有的：
46 /// M: Dynamically enable AMS logs @{
47 // Enable all debug log categories.
48 static boolean DEBUG_ALL = false;  //change to true

打开某一个debug开关，则单独修改对应的debug开关

最后build frameworks/base/services 模块即可


Contacts 打开log
```java
 public static final boolean FORCE_DEBUG =
            (SystemProperties.getInt(PROP_FORCE_DEBUG_KEY, 0) == 1);
```