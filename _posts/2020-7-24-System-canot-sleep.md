# 记一次pos机无法灭屏的bug

@(aosp/Android)

操作系统基于5.1.1
平台是: qualcomm

## bug的现象
### 测试提出的现象
 插充电器然后拔掉，休眠时间到不会休眠

### 我复测的现象

 去除USB掉之后, 发现依然会出现此类问题. 所以现在不能确定
一开始有人分反应只有在 Settings 界面有关. 但是经过多次测试, 即使在 Launcher 界面也会出现这个现象. 
因此和界面没有关系.

### 必现路径


### 分析原因
1. 通过dumpsys power 发现
```
POWER MANAGER (dumpsys power)

Power Manager State:
	// 省略掉不重要的东西
  mUserActivitySummary=0x1
  mRequestWaitForNegativeProximity=false
  mSandmanScheduled=false
  mSandmanSummoned=false
  mLowPowerModeEnabled=false
  mBatteryLevelLow=false
  mLastWakeTime=4919241 (8365 ms ago)
  mLastSleepTime=4902709 (24897 ms ago)
  mLastUserActivityTime=4922574 (5032 ms ago)  // 后面的值一直在改变
  mLastUserActivityTimeNoChangeLights=0 (4927606 ms ago)
  mLastInteractivePowerHintTime=4922574 (5032 ms ago)
  mLastScreenBrightnessBoostTime=0 (4927606 ms ago)
  mScreenBrightnessBoostInProgress=false
  mDisplayReady=true
  mHoldingWakeLockSuspendBlocker=false
  mHoldingDisplaySuspendBlocker=true

// 省略掉一部分
Wake Locks: size=0

Suspend Blockers: size=4
  PowerManagerService.WakeLocks: ref count=0
  PowerManagerService.Display: ref count=1
  PowerManagerService.Broadcasts: ref count=0
  PowerManagerService.WirelessChargerDetector: ref count=0

Display Power: state=ON
```
首先可以发现 还有一个 WakeLock 被 PowerManagerService.Display 所持有着。 这似乎合情合理的。 毕竟屏幕还在亮着。

再看系统的 mLastUserActivityTime 这个变， 从这个变量的名字上可以知道。 这个就是系统用来记录用户上次操作的时间。
这个时间的后面是 XXX ms ago。 这个可以在系统里知道。这个数字的含义是系统的当前时间和和用户上次操作的时间的差值。
在代码的 frameworks\base\services\core\java\com\android\server\power\PowerManagerService.java 
以下是 mLastUserActivityTime 是被赋值的时候。 可以看到他是被赋了 eventtime 的值。
``` JAVA
private boolean userActivityNoUpdateLocked(long eventTime, int event, int flags, int uid) {
    if (DEBUG_SPEW) {
        Slog.d(TAG, "userActivityNoUpdateLocked: eventTime=" + eventTime
                + ", event=" + event + ", flags=0x" + Integer.toHexString(flags)
                + ", uid=" + uid);
    }

    if (eventTime < mLastSleepTime || eventTime < mLastWakeTime
            || !mBootCompleted || !mSystemReady) {
        return false;
    }

    Trace.traceBegin(Trace.TRACE_TAG_POWER, "userActivity");
    try {
        if (eventTime > mLastInteractivePowerHintTime) {
            powerHintInternal(POWER_HINT_INTERACTION, 0);
            mLastInteractivePowerHintTime = eventTime;
        }

        mNotifier.onUserActivity(event, uid);

        if (mWakefulness == WAKEFULNESS_ASLEEP
                || mWakefulness == WAKEFULNESS_DOZING
                || (flags & PowerManager.USER_ACTIVITY_FLAG_INDIRECT) != 0) {
            return false;
        }

        if ((flags & PowerManager.USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS) != 0) {
            if (eventTime > mLastUserActivityTimeNoChangeLights
                    && eventTime > mLastUserActivityTime) {
                mLastUserActivityTimeNoChangeLights = eventTime;
                mDirty |= DIRTY_USER_ACTIVITY;
                return true;
            }
        } else {
            if (eventTime > mLastUserActivityTime) {
                mLastUserActivityTime = eventTime;
                mDirty |= DIRTY_USER_ACTIVITY;
                return true;
            }
        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_POWER);
    }
    return false;
}

```

这个时候我们需要打开 SPEW 的开关。 查看一下具体是哪一个调用了这个。
```
03-16 00:02:05.473   801  2448 D PowerManagerService1111: userActivityNoUpdateLocked: eventTime=1143410, event=1, flags=0x0, uid=1000
03-16 00:02:05.474   801  2448 D PowerManagerService1111: updateUserActivitySummaryLocked: mWakefulness=Awake, mUserActivitySummary=0x1, nextTimeout=1167410 (in 24000 ms)
03-16 00:02:05.474   801  2448 D PowerManagerService1111: updateDisplayPowerStateLocked: mDisplayReady=true, policy=3, mWakefulness=1, mWakeLockSummary=0x0, mUserActivitySummary=0x1, mBootCompleted=true, mScreenBrightnessBoostInProgress=false

```
从这个log里可以看到 event 的值是1， 那这个 1 是什么意思呢。 我们可以看到 userActivityNoUpdateLocked，被调用地方。 在同一个类里面的 boostScreenBrightnessInternal 方法里面，可以看到event 可以被赋值为PowerManager.USER_ACTIVITY_EVENT_OTHER。因此， event 的取值可以是以下三种
```
    /**
     * User activity event type: Unspecified event type.
     * @hide
     */
    @SystemApi
    public static final int USER_ACTIVITY_EVENT_OTHER = 0;

    /**
     * User activity event type: Button or key pressed or released.
     * @hide
     */
    @SystemApi
    public static final int USER_ACTIVITY_EVENT_BUTTON = 1;

    /**
     * User activity event type: Touch down, move or up.
     * @hide
     */
    @SystemApi
    public static final int USER_ACTIVITY_EVENT_TOUCH = 2;

```
USER_ACTIVITY_EVENT_OTHER  是非特别的的事件类型， USER_ACTIVITY_EVENT_BUTTON 是指按键被按下或者释放的事件。USER_ACTIVITY_EVENT_TOUCH 是屏幕触摸的事件类型，可以是down， move或者是up。也就是系统不断地接收到了 按键或者按钮不断地按下或者停止按下的事件。然后不断的改变了系统的记录用户操作的时间戳 userActivityNoUpdateLocked 。
从这里可以知道。 android 5.1的代码里， 目前只有一个方法 userActivityInternal 调用了userActivityNoUpdateLocked方法的时候event 的值是不确定的。 当然你可以加一个log
```
android.util.Slog.d(TAG, "" + android.util.log.getStackTraceString(new Throwable))
```
一路追踪可以发现。android_server_PowerManagerService_userActivity 最终是调用了这里。最后走到pokeUserActivity的 frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp.
于是我打开了InputFlinger里的InputDispatcher 里的所有log。 抓了一个log， 看了一下。没有发现异常（这里是错误的）
### 查看 input的 dump 文件
这个时候我们查看一下 adb shell dumpsys input ， 可以查看Input的dump文件。
```
INPUT MANAGER (dumpsys input)

Event Hub State:
  BuiltInKeyboardId: -2
  Devices:
    // 省略掉一部分 
    4: SX9310 Cap Touch
      Classes: 0x00000001
      Path: /dev/input/event2
      Descriptor: 690f01a0259cd417fdbc5cc545f132b3a6cafeaf
      Location: 
      ControllerNumber: 0
      UniqueId: 
      Identifier: bus=0x0018, vendor=0x0000, product=0x0000, version=0x0000
      KeyLayoutFile: /system/usr/keylayout/Generic.kl
      KeyCharacterMapFile: /system/usr/keychars/Generic.kcm
      ConfigurationFile: 
      HaveKeyboardLayoutOverlay: false
    // 省略掉一部分

Input Reader State:
  
  Device 4: SX9310 Cap Touch
    Generation: 8
    IsExternal: false
    Sources: 0x00000101
    KeyboardType: 1
    Keyboard Input Mapper:
      Parameters:
        HasAssociatedDisplay: false
        OrientationAware: false
        HandlesKeyRepeat: false
      KeyboardType: 1
      Orientation: 0
      KeyDowns: 1 keys currently down
      MetaState: 0x0
      DownTime: 6399626287000
  // 省略掉一部分

Input Dispatcher State:
  DispatchEnabled: 1
  DispatchFrozen: 0
  FocusedApplication: name='AppWindowToken{3a53dee1 token=Token{3bc02048 ActivityRecord{30ee7aeb u0 com.android.settings/.Settings t5}}}', dispatchingTimeout=5000.000ms
  FocusedWindow: name='Window{4fca645 u0 com.android.settings/com.android.settings.Settings}'
  TouchStates: <no displays touched>
  Windows:
    0: name='Window{3baf3ca9 u0 Heads Up}', displayId=0, paused=false, hasFocus=false, hasWallpaper=false, visible=false, canReceiveKeys=false, flags=0x01820328, type=0x000007de, layer=161000, frame=[0,0][720,0], scale=1.000000, touchableRegion=<empty>, inputFeatures=0x00000000, ownerPid=28708, ownerUid=10015, dispatchingTimeout=5000.000ms
    1: name='Window{3d13f071 u0 StatusBar}', displayId=0, paused=false, hasFocus=false, hasWallpaper=false, visible=true, canReceiveKeys=false, flags=0x81840048, type=0x000007d0, layer=151000, frame=[0,0][720,50], scale=1.000000, touchableRegion=[0,0][720,50], inputFeatures=0x00000000, ownerPid=28708, ownerUid=10015, dispatchingTimeout=5000.000ms
    2: name='Window{39a5f9c9 u0 KeyguardScrim}', displayId=0, paused=false, hasFocus=false, hasWallpaper=false, visible=false, canReceiveKeys=false, flags=0x01110900, type=0x000007ed, layer=131000, frame=[0,0][720,1280], scale=1.000000, touchableRegion=[0,0][720,1280], inputFeatures=0x00000000, ownerPid=801, ownerUid=1000, dispatchingTimeout=5000.000ms
    3: name='Window{4fca645 u0 com.android.settings/com.android.settings.Settings}', displayId=0, paused=false, hasFocus=true, hasWallpaper=false, visible=true, canReceiveKeys=true, flags=0x81810120, type=0x00000001, layer=21010, frame=[0,0][720,1280], scale=1.000000, touchableRegion=[0,0][720,1280], inputFeatures=0x00000000, ownerPid=5760, ownerUid=1000, dispatchingTimeout=5000.000ms
    4: name='Window{c1b2547 u0 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher}', displayId=0, paused=false, hasFocus=false, hasWallpaper=false, visible=false, canReceiveKeys=false, flags=0x81910120, type=0x00000001, layer=21005, frame=[0,0][720,1280], scale=1.000000, touchableRegion=[0,0][720,1280], inputFeatures=0x00000000, ownerPid=3909, ownerUid=10017, dispatchingTimeout=5000.000ms
    5: name='Window{2f7a3628 u0 com.android.systemui.ImageWallpaper}', displayId=0, paused=false, hasFocus=false, hasWallpaper=false, visible=true, canReceiveKeys=false, flags=0x00000318, type=0x000007dd, layer=21000, frame=[0,0][960,1280], scale=1.000000, touchableRegion=[0,0][960,1280], inputFeatures=0x00000000, ownerPid=28708, ownerUid=10015, dispatchingTimeout=5000.000ms
  MonitoringChannels:
    0: 'WindowManager (server)'
  RecentQueue: length=10
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10475), policyFlags=0x42000000, age=459.8ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10476), policyFlags=0x42000000, age=409.0ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10477), policyFlags=0x42000000, age=358.4ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10478), policyFlags=0x42000000, age=307.3ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10479), policyFlags=0x42000000, age=256.3ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10480), policyFlags=0x42000000, age=205.4ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10481), policyFlags=0x42000000, age=154.6ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10482), policyFlags=0x42000000, age=104.0ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10483), policyFlags=0x42000000, age=52.9ms
    KeyEvent(deviceId=4, source=0x00000101, action=0, flags=0x00000008, keyCode=261, scanCode=631, metaState=0x00000000, repeatCount=10484), policyFlags=0x42000000, age=2.5ms
  PendingEvent: <none>
  InboundQueue: <empty>
  ReplacedKeys: <empty>
  Connections:
    0: channelName='WindowManager (server)', windowName='monitor', status=NORMAL, monitor=true, inputPublisherBlocked=false
      OutboundQueue: <empty>
      WaitQueue: <empty>
    1: channelName='39a5f9c9 KeyguardScrim (server)', windowName='Window{39a5f9c9 u0 KeyguardScrim}', status=NORMAL, monitor=false, inputPublisherBlocked=false
      OutboundQueue: <empty>
      WaitQueue: <empty>
    2: channelName='2f7a3628 com.android.systemui.ImageWallpaper (server)', windowName='Window{2f7a3628 u0 com.android.systemui.ImageWallpaper}', status=NORMAL, monitor=false, inputPublisherBlocked=false
      OutboundQueue: <empty>
      WaitQueue: <empty>
    3: channelName='4fca645 com.android.settings/com.android.settings.Settings (server)', windowName='Window{4fca645 u0 com.android.settings/com.android.settings.Settings}', status=NORMAL, monitor=false, inputPublisherBlocked=false
      OutboundQueue: <empty>
      WaitQueue: <empty>
    4: channelName='c1b2547 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)', windowName='Window{c1b2547 u0 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher}', status=NORMAL, monitor=false, inputPublisherBlocked=false
      OutboundQueue: <empty>
      WaitQueue: <empty>
    5: channelName='3d13f071 StatusBar (server)', windowName='Window{3d13f071 u0 StatusBar}', status=NORMAL, monitor=false, inputPublisherBlocked=false
      OutboundQueue: <empty>
      WaitQueue: <empty>
    6: channelName='3baf3ca9 Heads Up (server)', windowName='Window{3baf3ca9 u0 Heads Up}', status=NORMAL, monitor=false, inputPublisherBlocked=false
      OutboundQueue: <empty>
      WaitQueue: <empty>
  AppSwitch: not pending
  Configuration:
    KeyRepeatDelay: 50.0ms
    KeyRepeatTimeout: 500.0ms
```
从RecentQueue里面我们可以发现， 最后系统执行的时间是KeyEvent，这个就很有问题了，因为我最后是操作了屏幕， 并没有操作某一个按键，所以他应该是 MotionEvent 事件。 从这里我们看到上报KeyEvent的是设备号 deviceId =4。 再从上面的DeviceID 可以看到这个是一个SX9310 Cap Touch。 在InputReader里可以看到KeyDowns 这里说明为， 有一个键是Down的状态。所以猜测很可能是这个Sensor 一直上报事件导致的。
我们可以吧这个器件的 event节点删除/dev/input/event2。 再次进行多次测试， 也没有测试来不灭屏的现象了。
然后我们看看没删除event2之前的系统的 getevent -lt ， 可以查看一下但是没有事件的输出。
然后我们在在系统编译的时候去掉SAR， 然后替换掉bootimage。再次测试的时候，测试了多次， 并没有出现不灭屏幕的情况。因此基本上可以确定是 SAR sensor的问题。

### 回过头看InputFlinger的log
在之前的我抓去了 InputFlinger 的log。我并没有发现问题， 但其实是我看错了。我们在仔细的看一下这个log
```
03-16 00:02:04.967   801  2448 D PowerManagerService1111: userActivityNoUpdateLocked: eventTime=1142903, event=1, flags=0x0, uid=1000
03-16 00:02:04.967   801  2448 D PowerManagerService1111: updateUserActivitySummaryLocked: mWakefulness=Awake, mUserActivitySummary=0x1, nextTimeout=1166903 (in 24000 ms)
03-16 00:02:04.967   801  2448 D PowerManagerService1111: updateDisplayPowerStateLocked: mDisplayReady=true, policy=3, mWakefulness=1, mWakeLockSummary=0x0, mUserActivitySummary=0x1, mBootCompleted=true, mScreenBrightnessBoostInProgress=false
03-16 00:02:04.967   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:04.967   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:04.967   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:04.967   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:04.967   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:04.968   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:04.968   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:04.968   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34197, handled=false
03-16 00:02:04.968   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:04.969   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34196, handled=false
03-16 00:02:04.969   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14661, policyFlags=0x42000000
03-16 00:02:04.969   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.017   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.017   801  2448 D InputDispatcher: dispatchKey - eventTime=1142954240032, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14662, downTime=395357128000
03-16 00:02:05.018   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.018   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.018   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.018   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.018   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.018   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.018   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.024   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34199, handled=false
03-16 00:02:05.024   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.024   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34198, handled=false
03-16 00:02:05.025   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14662, policyFlags=0x42000000
03-16 00:02:05.025   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.068   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.068   801  2448 D InputDispatcher: dispatchKey - eventTime=1143004930917, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14663, downTime=395357128000
03-16 00:02:05.068   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.068   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.068   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.068   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.068   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.068   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.068   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.069   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34201, handled=false
03-16 00:02:05.069   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.070   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34200, handled=false
03-16 00:02:05.070   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14663, policyFlags=0x42000000
03-16 00:02:05.070   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.119   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.119   801  2448 D InputDispatcher: dispatchKey - eventTime=1143056100917, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14664, downTime=395357128000
03-16 00:02:05.119   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.119   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.119   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.119   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.120   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.120   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.120   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.121   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34203, handled=false
03-16 00:02:05.121   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.121   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34202, handled=false
03-16 00:02:05.121   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14664, policyFlags=0x42000000
03-16 00:02:05.121   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.142   801  3251 E LocSvc_libulp: I/===> int ulp_msg_process_system_update(UlpSystemEvent) line 1455 
03-16 00:02:05.142   801  3251 E LocSvc_libulp: I/int ulp_msg_process_system_update(UlpSystemEvent): systemEvent:5 
03-16 00:02:05.142   801  3251 E LocSvc_libulp: I/===> int ulp_brain_process_system_update(UlpSystemEvent) line 2352 
03-16 00:02:05.142   801  3251 E LocSvc_libulp: I/===> int ulp_brain_select_providers() line 333 
03-16 00:02:05.142   801  3251 E LocSvc_libulp: I/===> int ulp_brain_turn_onoff_quipc_provider() line 700 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> int ulp_quipc_stop_engine() line 235 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> bool ulp_quipc_engine_running() line 60 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> int ulp_brain_turn_onoff_gnss_provider() line 595 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> int ulp_gnss_stop_engine() line 262 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> bool ulp_quipc_engine_running() line 60 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> int ulp_brain_turn_onoff_gnp_provider() line 496 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> bool ulp_gnp_engine_running() line 59 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> int ulp_gnp_stop_engine() line 195 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> bool ulp_gnp_engine_running() line 59 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> bool ulp_gnp_engine_running() line 59 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> int ulp_brain_turn_onoff_zpp_provider() line 437 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> int ulp_zpp_start_engine() line 98 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> bool ulp_zpp_engine_running() line 62 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/===> int ulp_msg_process_start_req() line 447 
03-16 00:02:05.143   801  3251 E LocSvc_libulp: I/int ulp_msg_process_start_req(), at ulp state = 1
03-16 00:02:05.143   801  2860 E LocSvc_api_v02: I/---> locClientSendReq line 2065 QMI_LOC_GET_BEST_AVAILABLE_POSITION_REQ_V02
03-16 00:02:05.144   801  2963 E LocSvc_ApiV02: I/<--- void globalRespCb(locClientHandleType, uint32_t, locClientRespIndUnionType, void*) line 111 QMI_LOC_GET_BEST_AVAILABLE_POSITION_REQ_V02
03-16 00:02:05.144   801  3251 E LocSvc_libulp: I/===> int ulp_brain_process_zpp_position_report(loc_sess_status, LocPosTechMask, const UlpLocation*) line 1463 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/int ulp_brain_process_zpp_position_report(loc_sess_status, LocPosTechMask, const UlpLocation*), report ZPP position to providers,report_position = 0
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_brain_select_providers() line 333 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_brain_turn_onoff_quipc_provider() line 700 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_quipc_stop_engine() line 235 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> bool ulp_quipc_engine_running() line 60 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_brain_turn_onoff_gnss_provider() line 595 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_gnss_stop_engine() line 262 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> bool ulp_quipc_engine_running() line 60 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_brain_turn_onoff_gnp_provider() line 496 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> bool ulp_gnp_engine_running() line 59 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_gnp_stop_engine() line 195 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> bool ulp_gnp_engine_running() line 59 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> bool ulp_gnp_engine_running() line 59 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_brain_turn_onoff_zpp_provider() line 437 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> int ulp_zpp_stop_engine() line 174 
03-16 00:02:05.145   801  3251 E LocSvc_libulp: I/===> bool ulp_zpp_engine_running() line 62 
03-16 00:02:05.159   262   262 I SurfaceFlinger: FPS: 7
03-16 00:02:05.170   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.170   801  2448 D InputDispatcher: dispatchKey - eventTime=1143106812271, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14665, downTime=395357128000
03-16 00:02:05.170   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.170   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.170   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.170   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.170   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.171   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.171   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.171   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34205, handled=false
03-16 00:02:05.171   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.172   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34204, handled=false
03-16 00:02:05.172   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14665, policyFlags=0x42000000
03-16 00:02:05.172   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.220   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.220   801  2448 D InputDispatcher: dispatchKey - eventTime=1143157005500, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14666, downTime=395357128000
03-16 00:02:05.220   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.220   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.220   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.220   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.221   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.221   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.221   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.221   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34207, handled=false
03-16 00:02:05.222   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.222   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34206, handled=false
03-16 00:02:05.222   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14666, policyFlags=0x42000000
03-16 00:02:05.222   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.270   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.270   801  2448 D InputDispatcher: dispatchKey - eventTime=1143207235500, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14667, downTime=395357128000
03-16 00:02:05.270   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.270   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.271   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.271   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.271   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.271   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.271   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.271   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34209, handled=false
03-16 00:02:05.271   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.272   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34208, handled=false
03-16 00:02:05.272   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14667, policyFlags=0x42000000
03-16 00:02:05.272   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.320   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.320   801  2448 D InputDispatcher: dispatchKey - eventTime=1143257433365, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14668, downTime=395357128000
03-16 00:02:05.321   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.321   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.321   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.321   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.321   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.321   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.321   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.322   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34211, handled=false
03-16 00:02:05.322   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.323   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34210, handled=false
03-16 00:02:05.323   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14668, policyFlags=0x42000000
03-16 00:02:05.323   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.371   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.371   801  2448 D InputDispatcher: dispatchKey - eventTime=1143307931386, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14669, downTime=395357128000
03-16 00:02:05.371   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.371   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.371   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.371   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.372   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.372   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.372   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.372   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34213, handled=false
03-16 00:02:05.372   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.373   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34212, handled=false
03-16 00:02:05.373   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14669, policyFlags=0x42000000
03-16 00:02:05.373   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.422   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.422   801  2448 D InputDispatcher: dispatchKey - eventTime=1143359032219, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14670, downTime=395357128000
03-16 00:02:05.422   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.422   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.422   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.422   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.423   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.423   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.423   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.424   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34215, handled=false
03-16 00:02:05.424   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.425   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34214, handled=false
03-16 00:02:05.425   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14670, policyFlags=0x42000000
03-16 00:02:05.425   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.473   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.473   801  2448 D InputDispatcher: dispatchKey - eventTime=1143410186646, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14671, downTime=395357128000
03-16 00:02:05.473   801  2448 D PowerManagerService1111: userActivityNoUpdateLocked: eventTime=1143410, event=1, flags=0x0, uid=1000
03-16 00:02:05.474   801  2448 D PowerManagerService1111: updateUserActivitySummaryLocked: mWakefulness=Awake, mUserActivitySummary=0x1, nextTimeout=1167410 (in 24000 ms)
03-16 00:02:05.474   801  2448 D PowerManagerService1111: updateDisplayPowerStateLocked: mDisplayReady=true, policy=3, mWakefulness=1, mWakeLockSummary=0x0, mUserActivitySummary=0x1, mBootCompleted=true, mScreenBrightnessBoostInProgress=false

```

在这里我选取了一个循环的，因为他是两次PowerManagerService， 去改变了 UserLastActivityTime。因此我们可以选择两次PowerManagerService 之间的log
在这里我们可以看到 repeatCount=14670 这明显是不正常的。同时我们也可以看到，这个log也是循环的出现的。 我截取一个循环之内的
```
03-16 00:02:05.422   801  2448 D InputDispatcher: dispatchKey - eventTime=1143359032219, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14670, downTime=395357128000
03-16 00:02:05.422   801  2448 D InputDispatcher: findFocusedWindow finished: injectionResult=0, timeSpentWaitingForApplication=0.0ms
03-16 00:02:05.422   801  2448 D InputDispatcher: dispatchEventToCurrentInputTargets
03-16 00:02:05.422   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ prepareDispatchCycle - flags=0x00000101, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.422   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.423   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ prepareDispatchCycle - flags=0x00000100, xOffset=0.000000, yOffset=0.000000, scaleFactor=1.000000, pointerIds=0x0
03-16 00:02:05.423   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.423   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.424   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ finishDispatchCycle - seq=34215, handled=false
03-16 00:02:05.424   801  2448 D InputDispatcher: channel 'WindowManager (server)' ~ startDispatchCycle
03-16 00:02:05.425   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ finishDispatchCycle - seq=34214, handled=false
03-16 00:02:05.425   801  2448 D InputDispatcher: Unhandled key event: Skipping unhandled key event processing since this is not an initial down.  keyCode=261, action=0, repeatCount=14670, policyFlags=0x42000000
03-16 00:02:05.425   801  2448 D InputDispatcher: channel '1e294a8 com.cyanogenmod.trebuchet/com.android.launcher3.Launcher (server)' ~ startDispatchCycle
03-16 00:02:05.473   801  2448 D InputDispatcher: Resetting ANR timeouts.
03-16 00:02:05.473   801  2448 D InputDispatcher: dispatchKey - eventTime=1143410186646, deviceId=4, source=0x101, policyFlags=0x42000000, action=0x0, flags=0x8, keyCode=0x105, scanCode=0x277, metaState=0x0, repeatCount=14671, downTime=395357128000

```


我整理出大概的流程如下
```
findFocusedWindowTargetsLocked
dispatchEventLocked
prepareDispatchCycleLocked
startDispatchCycleLocked
prepareDispatchCycleLocked
startDispatchCycleLocked
resetANRTimeoutsLocked
finishDispatchCycleLocked
startDispatchCycleLocked
finishDispatchCycleLocked
afterKeyEventLockedInterruptible
startDispatchCycleLocked
resetANRTimeoutsLocked
dispatchKeyLocked
```


以下是上面的 log 一次循环的的大概流程.
```
dispatchOnceInnerLocked
    resetANRTimeoutsLocked
    dispatchKeyLocked
        logOutboundKeyDetailsLocked
        findFocusedWindowTargetsLocked
        dispatchEventLocked
            prepareDispatchCycleLocked
                enqueueDispatchEntriesLocked
                    startDispatchCycleLocked
            prepareDispatchCycleLocked
                enqueueDispatchEntriesLocked
                    startDispatchCycleLocked
    releasePendingEventLocked
        resetANRTimeoutsLocked

// 似乎在这里没有看到衔接的方法， 也就是明显的调用链
registerInputChannel    
    handleReceiveCallback
        finishDispatchCycleLocked
            onDispatchCycleFinishedLocked
                doDispatchCycleFinishedLockedInterruptible
                    startDispatchCycleLocked
        finishDispatchCycleLocked
            onDispatchCycleFinishedLocked
                doDispatchCycleFinishedLockedInterruptible
                    afterKeyEventLockedInterruptible
                        startDispatchCycleLocked
```

## 解决方案

### InputFlinger 里处理
在 InputReader 方法里
```
// frameworks/native/services/inputflinger/InputReader.cpp
void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t keyCode,
        int32_t scanCode, uint32_t policyFlags) {
	// Down 事件的处理
    if (down) {
        // Rotate key codes according to orientation if needed.
        if (mParameters.orientationAware && mParameters.hasAssociatedDisplay) {
            keyCode = rotateKeyCode(keyCode, mOrientation);
        }

        // Add key down.
        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex >= 0) {
            // key repeat, be sure to use same keycode as before in case of rotation
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
        } else {
            // key down
            if ((policyFlags & POLICY_FLAG_VIRTUAL)
                    && mContext->shouldDropVirtualKey(when,
                            getDevice(), keyCode, scanCode)) {
                return;
            }

            mKeyDowns.push();
            KeyDown& keyDown = mKeyDowns.editTop();
            keyDown.keyCode = keyCode;
            keyDown.scanCode = scanCode;
            // 对Sar 的键值进行特殊的处理, 添加不要重复的Flag
            if(keyDown.keyCode == AKEYCODE_SARSENSOR){
                policyFlags |= POLICY_FLAG_DISABLE_KEY_REPEAT;
            }
        }
        mDownTime = when;
    }
    //省略代码
}

```

### 添加配置节点
手动添加节点。
```
echo "keyboard.handlesKeyRepeat = 1" > /system/usr/idc/SX9310_Cap_Touch.idc
```
添加 SAR的 一个数据节点文件， 并且向里面写 keyboard.handlesKeyRepeat = 1 数据.
这个方法是基于， InputReader 会有读取Configure的行为。在 InputReader 里的有一个方法 configureParameters。
这个节点不支持空格， 以下划线替代。
``` cpp
// frameworks/native/services/inputflinger/InputReader.cpp
void KeyboardInputMapper::configureParameters() {
    mParameters.orientationAware = false;
    getDevice()->getConfiguration().tryGetProperty(String8("keyboard.orientationAware"),
            mParameters.orientationAware);

    mParameters.hasAssociatedDisplay = false;
    if (mParameters.orientationAware) {
        mParameters.hasAssociatedDisplay = true;
    }

    mParameters.handlesKeyRepeat = false;
    getDevice()->getConfiguration().tryGetProperty(String8("keyboard.handlesKeyRepeat"),
            mParameters.handlesKeyRepeat);
}

```
他在读取这个值之后， 这个上传的KeyEvent一样会产生不重复的效果。 
在代码里。 我们可以再 frameworks/base/data/keyboards/ 路径下添加SX9310_Cap_Touch.idc 文件， 文件的内容就是keyboard.handlesKeyRepeat = 1。 这样就可以了。



### 添加数据节点

sar 是一直报键值的, 报的还是 KeyEvent 类型的键值, 这类键值事件上报上来了, 必须要做处理. 
SAR 这种一直报键值的方式是不对的，个人觉得最好的办法是： 因为 SAR 是只给 modem 工作的一种 sensor， 工模里需要该键值来测试 SAR 的 sensor 是否正常工作。 建议的方式， 直接写节点。然后 app 去读取这个节点， 就可以达到检测sar 传感器的了。这样避免了形成系统的 event 事件， 然后在封装成 KeyEvent , 再给 InputFlinger， InputFlinger 再派发出去到某一个界面。这种设计太过繁琐。可以直接写成一个数据节点，然后工模的 App 读取这个数值。 这样可以更直接一点。


## 其他反应情况
### 为什么有时候又是正常灭屏的
这种情况，抓取log， 可以发现， 并没有发生repeat count的行为



### 为什么修改灭屏时间又容易复现
目前经过多次测试， 没有发现修改灭屏时间复现该问题。 个人觉得可能是测试的时候移动了该测试设备。


### 和插USB没关系
因为插USB移动了pos 机器。导致sar 工作了。 表现就像是插入了 USB，灭屏 功能才会异常的。

### 灭屏， 也报sar的情况， 但不会亮屏
这个时候他是没有窗口了。 它将含有INPUT_FEATURE_DISABLE_USER_ACTIVITY ， 所以直接回return 掉，不会继续向下传输到PowerManagerService

``` cpp
void InputDispatcher::pokeUserActivityLocked(const EventEntry* eventEntry) {
    if (mFocusedWindowHandle != NULL) {
        const InputWindowInfo* info = mFocusedWindowHandle->getInfo();
        if (info->inputFeatures & InputWindowInfo::INPUT_FEATURE_DISABLE_USER_ACTIVITY) {
#if DEBUG_DISPATCH_CYCLE
            ALOGD("Not poking user activity: disabled by window '%s'.", info->name.string());
#endif
            return;
        }
    }
    // 省略掉部分代码
}

```

### 
```
getevent 之后抬起手机， event2的确有事件上报

[    4605.448133] /dev/input/event2: EV_SYN       SYN_REPORT           00000000            
[    4607.069567] /dev/input/event2: EV_KEY       0277                 UP                  
[    4607.069567] /dev/input/event2: EV_SYN       SYN_REPORT           00000000            
[    4614.606782] /dev/input/event2: EV_KEY       0277                 DOWN                
[    4614.606782] /dev/input/event2: EV_SYN       SYN_REPORT           00000000            
[    4615.687440] /dev/input/event2: EV_KEY       0277                 UP                  
[    4615.687440] /dev/input/event2: EV_SYN       SYN_REPORT           00000000            
[    4616.498241] /dev/input/event2: EV_KEY       0277                 DOWN                
[    4616.498241] /dev/input/event2: EV_SYN       SYN_REPORT           00000000            
[    4618.720999] /dev/input/event2: EV_KEY       0277                 UP                  
[    4618.720999] /dev/input/event2: EV_SYN       SYN_REPORT           00000000            
[    4619.831867] /dev/input/event2: EV_KEY       0277                 DOWN                
[    4619.831867] /dev/input/event2: EV_SYN       SYN_REPORT           00000000  
```
Sar sensor， 在这部手机上的表现是， 当手机的屏幕朝上放置的时候，拿起手机是报up键值的。 放下手机是报的down键值。
可以看到， 这里手机一直处于down状态。


一直down. 没其他事件刷新 因此触发repeat机制， 一直分发相同的事件