---
layout:     post
title:      "Notification分析"
subtitle:   "Android Notification分析"
date:       2017-05-14 00:10:40
author:     "Mathew"
catalog: true
header-img: "img/post-bg-2017.jpg"
tags:
    - Android
    - Notification
---

# Notification

[TOC]

  虽然网络上很大神对Android 的Notification都介绍了， 写的很详细, 很深入. 在Android N上Google对Notification进行了非常大的改造(同时有相关的中文和英文[文档](https://developer.android.com/guide/topics/ui/notifiers/notifications.html)) 但是我还是想以我的方式, 以我的理解来解读一下Android N 上面的Notification. 我会从以下几个方面来讲解Notification. 
  

## 从界面上认识Notification

 首先 Notification是分种类的, 一共有以下几个种类浮动通知(Heads-up Notifications), 锁屏通知Lock Screen Notifications, Bundling Notifications, Peeking Notifications. 
第二 Notification是分优先级(priority)的一共有五个等级; 分别是PRIORITY_LOW， PRIORITY_MIN, PRIORITY_DEFAULT, PRIORITY_HIGH, PRIORITY_MAX, 依次是-2， -1， 0， 1， 2. 一般不指定优先级的话, 默认的是为零, 在Notification的运行当中, 会被封装成其他的类, 同样优先级会被转换成其他的数字. 
第三 Notification有不同的样式. 有
```
BigPictureStyle
BigTextStyle
DecoratedCustomViewStyle
DecoratedMediaCustomViewStyle
InboxStyle
MediaStyle
MessagingStyle
```

第四: Notification 的有一个defaults属性,可以设置Notification的通知灯, 震动, 声音. 默认情况是所有的都包含


第五: Notification的color属性将会给通知的small icon染上颜色



```
.
|-- CalendarTracker.java
|-- ConditionProviders.java
|-- CountdownConditionProvider.java
|-- EventConditionProvider.java
|-- GlobalSortKeyComparator.java
|-- ImportanceExtractor.java
|-- ManagedServices.java
|-- NotificationComparator.java
|-- NotificationDelegate.java
|-- NotificationIntrusivenessExtractor.java
|-- NotificationManagerInternal.java
|-- NotificationManagerService.java
|-- NotificationRecord.java
|-- NotificationSignalExtractor.java
|-- NotificationUsageStats.java
|-- PriorityExtractor.java
|-- PropConfig.java
|-- RankingConfig.java
|-- RankingHandler.java
|-- RankingHelper.java
|-- RankingReconsideration.java
|-- RateEstimator.java
|-- ScheduleCalendar.java
|-- ScheduleConditionProvider.java
|-- SystemConditionProviderService.java
|-- ValidateNotificationPeople.java
|-- VisibilityExtractor.java
|-- ZenLog.java
|-- ZenModeConditions.java
|-- ZenModeFiltering.java
`-- ZenModeHelper.java

0 directories, 31 files
```

```
Notification.aidl
Notification.java
NotificationManager.aidl
NotificationManager.java

```

```
.
|-- MediaProjectionPermissionActivity.java
|-- NotificationPlayer.java
`-- RingtonePlayer.java
```


## 从代码上认识Notification

```java
public void notify(int id, Notification notification)
```

```java
public void notify(String tag, int id, Notification notification)
```

```java
public void notifyAsUser(String tag, int id, Notification notification, UserHandle user)
```

```java
public void notifyAsUser(String tag, int id, Notification notification, UserHandle user)
{
    int[] idOut = new int[1];
    INotificationManager service = getService(); //获取NotificationManagerService
    String pkg = mContext.getPackageName();
    // Fix the notification as best we can.
    Notification.addFieldsFromContext(mContext, notification);  //主要是向notification加入应用信息和用户ID
    if (notification.sound != null) {
        notification.sound = notification.sound.getCanonicalUri(); //获取通知声音的实际地址
        if (StrictMode.vmFileUriExposureEnabled()) {
            notification.sound.checkFileUriExposed("Notification.sound");
        }
    }
    fixLegacySmallIcon(notification, pkg); //修复一些icon问题,如果没有就给他指定一个
    if (mContext.getApplicationInfo().targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1) {
        if (notification.getSmallIcon() == null) {
            throw new IllegalArgumentException("Invalid notification (no valid small icon): "
                    + notification);
        }
    }
    if (localLOGV) Log.v(TAG, pkg + ": notify(" + id + ", " + notification + ")");
    final Notification copy = Builder.maybeCloneStrippedForDelivery(notification);
    try {
        service.enqueueNotificationWithTag(pkg, mContext.getOpPackageName(), tag, id,
                copy, idOut, user.getIdentifier());
        if (id != idOut[0]) {
            Log.w(TAG, "notify: id corrupted: sent " + id + ", got back " + idOut[0]);
        }
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```



```java
public static Notification maybeCloneStrippedForDelivery(Notification n) {
    String templateClass = n.extras.getString(EXTRA_TEMPLATE);

    // Only strip views for known Styles because we won't know how to
    // re-create them otherwise.
    if (!TextUtils.isEmpty(templateClass)
            && getNotificationStyleClass(templateClass) == null) {
        return n;
    }

    // Only strip unmodified BuilderRemoteViews.
    boolean stripContentView = n.contentView instanceof BuilderRemoteViews &&
            n.extras.getInt(EXTRA_REBUILD_CONTENT_VIEW_ACTION_COUNT, -1) ==
                    n.contentView.getSequenceNumber();
    boolean stripBigContentView = n.bigContentView instanceof BuilderRemoteViews &&
            n.extras.getInt(EXTRA_REBUILD_BIG_CONTENT_VIEW_ACTION_COUNT, -1) ==
                    n.bigContentView.getSequenceNumber();
    boolean stripHeadsUpContentView = n.headsUpContentView instanceof BuilderRemoteViews &&
            n.extras.getInt(EXTRA_REBUILD_HEADS_UP_CONTENT_VIEW_ACTION_COUNT, -1) ==
                    n.headsUpContentView.getSequenceNumber();

    // Nothing to do here, no need to clone.
    if (!stripContentView && !stripBigContentView && !stripHeadsUpContentView) {
        return n;
    }

    Notification clone = n.clone();
    if (stripContentView) {
        clone.contentView = null;
        clone.extras.remove(EXTRA_REBUILD_CONTENT_VIEW_ACTION_COUNT);
    }
    if (stripBigContentView) {
        clone.bigContentView = null;
        clone.extras.remove(EXTRA_REBUILD_BIG_CONTENT_VIEW_ACTION_COUNT);
    }
    if (stripHeadsUpContentView) {
        clone.headsUpContentView = null;
        clone.extras.remove(EXTRA_REBUILD_HEADS_UP_CONTENT_VIEW_ACTION_COUNT);
    }
    return clone;
}
```

```java
void enqueueNotificationInternal(final String pkg, final String opPkg, final int callingUid,
        final int callingPid, final String tag, final int id, final Notification  notification,
        int[] idOut, int incomingUserId) {
    if (DBG) {
        Slog.v(TAG, "enqueueNotificationInternal: pkg=" + pkg + " id=" + id
                + " notification=" + notification);
    }
    checkCallerIsSystemOrSameApp(pkg); //检查一下通知是否由系统发送过来的还是同一个通知发送过来的
    //是否系统发送过来的主要是通过Linux分配给进程的UID来判断
    final boolean isSystemNotification = isUidSystem(callingUid) || ("android".equals(pkg)); //是否是系统的通知
    final boolean isNotificationFromListener = mListeners.isListenerPackage(pkg);

    final int userId = ActivityManager.handleIncomingUser(callingPid,
            callingUid, incomingUserId, true, false, "enqueueNotification", pkg);// 获取当前的用户id
    final UserHandle user = new UserHandle(userId);

    // Fix the notification as best we can.
    try {
        final ApplicationInfo ai = getContext().getPackageManager().getApplicationInfoAsUser(
                pkg, PackageManager.MATCH_DEBUG_TRIAGED_MISSING,
                (userId == UserHandle.USER_ALL) ? UserHandle.USER_SYSTEM : userId);
        Notification.addFieldsFromContext(ai, userId, notification); //向notification里添加应用信息, 和当前用户的信息
    } catch (NameNotFoundException e) {
        Slog.e(TAG, "Cannot create a context for sending app", e);
        return;
    }

    mUsageStats.registerEnqueuedByApp(pkg);

    // Limit the number of notifications that any given package except the android
    // package or a registered listener can enqueue.  Prevents DOS attacks and deals with leaks.
    //除了系统的通知或者是来自于注册监听的通知, 限制所有的通知数量, 防止DOS攻击和处理泄露
    if (!isSystemNotification && !isNotificationFromListener) {
        synchronized (mNotificationList) {
            final float appEnqueueRate = mUsageStats.getAppEnqueueRate(pkg); //统计当前包所有的通知的数量
            if (appEnqueueRate > mMaxPackageEnqueueRate) {
                mUsageStats.registerOverRateQuota(pkg);
                final long now = SystemClock.elapsedRealtime();
                if ((now - mLastOverRateLogTime) > MIN_PACKAGE_OVERRATE_LOG_INTERVAL) {
                    Slog.e(TAG, "Package enqueue rate is " + appEnqueueRate
                            + ". Shedding events. package=" + pkg);
                    mLastOverRateLogTime = now;
                }
                return;
            }

            int count = 0;
            final int N = mNotificationList.size();
            for (int i=0; i<N; i++) {
                final NotificationRecord r = mNotificationList.get(i);
                if (r.sbn.getPackageName().equals(pkg) && r.sbn.getUserId() == userId) {
                    if (r.sbn.getId() == id && TextUtils.equals(r.sbn.getTag(), tag)) {
                        break;  // Allow updating existing notification
                    }
                    count++;
                    if (count >= MAX_PACKAGE_NOTIFICATIONS) {
                        mUsageStats.registerOverCountQuota(pkg);
                        Slog.e(TAG, "Package has already posted " + count
                                + " notifications.  Not showing more.  package=" + pkg);
                        return;
                    }
                }
            }
        }
    }

    if (pkg == null || notification == null) {
        throw new IllegalArgumentException("null not allowed: pkg=" + pkg
                + " id=" + id + " notification=" + notification);
    }

    // Whitelist pending intents.
    if (notification.allPendingIntents != null) {
        final int intentCount = notification.allPendingIntents.size();
        if (intentCount > 0) {
            final ActivityManagerInternal am = LocalServices
                    .getService(ActivityManagerInternal.class);
            final long duration = LocalServices.getService(
                    DeviceIdleController.LocalService.class).getNotificationWhitelistDuration();
            for (int i = 0; i < intentCount; i++) {
                PendingIntent pendingIntent = notification.allPendingIntents.valueAt(i);
                if (pendingIntent != null) {
                    am.setPendingIntentWhitelistDuration(pendingIntent.getTarget(), duration);
                }
            }
        }
    }

    // Sanitize inputs
    //审查一下当前通知的优先级, 如果当前优先级小于通知的最小优先级则赋予最小的优先级, 如果当前通知的优先级大于通知的最大优先级则赋予他最大的优先级
    notification.priority = clamp(notification.priority, Notification.PRIORITY_MIN,
            Notification.PRIORITY_MAX);

    // setup local book-keeping
    //将一个Notification封装成成一个StatusBarNotification.
    final StatusBarNotification n = new StatusBarNotification(
            pkg, opPkg, id, tag, callingUid, callingPid, 0, notification,
            user);
    //同时将StatusBarNotication记录下来
    final NotificationRecord r = new NotificationRecord(getContext(), n);
    //最后以Handler的post形式执行一个线程
    mHandler.post(new EnqueueNotificationRunnable(userId, r));

    idOut[0] = id;
}
```



```java
{
    synchronized (mNotificationList) {
        final StatusBarNotification n = r.sbn;
        if (DBG) Slog.d(TAG, "EnqueueNotificationRunnable.run for: " + n.getKey());
        NotificationRecord old = mNotificationsByKey.get(n.getKey());
        if (old != null) {
            // Retain ranking information from previous record
            r.copyRankingInformation(old);
        }

        final int callingUid = n.getUid();
        final int callingPid = n.getInitialPid();
        final Notification notification = n.getNotification();
        final String pkg = n.getPackageName();
        final int id = n.getId();
        final String tag = n.getTag();
        final boolean isSystemNotification = isUidSystem(callingUid) ||
                ("android".equals(pkg));

        // Handle grouped notifications and bail out early if we
        // can to avoid extracting signals.
        //Notification以群组的方式显示, 就是Android N的Bundled notifications
        handleGroupedNotificationLocked(r, old, callingUid, callingPid);

        // This conditional is a dirty hack to limit the logging done on
        //     behalf of the download manager without affecting other apps.
        //这段解释见Dirty Hack参考文献
        //主要是为了防止DownloadManager影响其他的应用, 这是临时的方法解决这个问题. 
        //那么DownloadManager是怎么影响其他的应用呢？
        if (!pkg.equals("com.android.providers.downloads")
                || Log.isLoggable("DownloadManager", Log.VERBOSE)) {
            int enqueueStatus = EVENTLOG_ENQUEUE_STATUS_NEW;
            if (old != null) {
                enqueueStatus = EVENTLOG_ENQUEUE_STATUS_UPDATE;
            }
            EventLogTags.writeNotificationEnqueue(callingUid, callingPid,
                    pkg, id, tag, userId, notification.toString(),
                    enqueueStatus);
        }

        mRankingHelper.extractSignals(r);

        final boolean isPackageSuspended = isPackageSuspendedForUser(pkg, callingUid);

        // blocked apps
        //阻止某些app的通知, 如果的通知的重要性为NONE, 当前应用不被允许弹出通知, 或者当前的应用没有权限, 
        //同时不是系统的通知
        if (r.getImportance() == NotificationListenerService.Ranking.IMPORTANCE_NONE
                || !noteNotificationOp(pkg, callingUid) || isPackageSuspended) {
            if (!isSystemNotification) {
                if (isPackageSuspended) {
                    Slog.e(TAG, "Suppressing notification from package due to package "
                            + "suspended by administrator.");
                    mUsageStats.registerSuspendedByAdmin(r);
                } else {
                    Slog.e(TAG, "Suppressing notification from package by user request.");
                    mUsageStats.registerBlocked(r);
                }
                return;
            }
        }

        // tell the ranker service about the notification
        if (mRankerServices.isEnabled()) {
            mRankerServices.onNotificationEnqueued(r);
            // TODO delay the code below here for 100ms or until there is an answer
        }


        int index = indexOfNotificationLocked(n.getKey());
        if (index < 0) {
            mNotificationList.add(r);
            mUsageStats.registerPostedByApp(r);
        } else {
            old = mNotificationList.get(index);
            mNotificationList.set(index, r);
            mUsageStats.registerUpdatedByApp(r, old);
            // Make sure we don't lose the foreground service state.
            notification.flags |=
                    old.getNotification().flags & Notification.FLAG_FOREGROUND_SERVICE;
            r.isUpdate = true;
        }

        mNotificationsByKey.put(n.getKey(), r);

        // Ensure if this is a foreground service that the proper additional
        // flags are set.
        if ((notification.flags & Notification.FLAG_FOREGROUND_SERVICE) != 0) {
            notification.flags |= Notification.FLAG_ONGOING_EVENT
                    | Notification.FLAG_NO_CLEAR;
        }

        applyZenModeLocked(r);
        mRankingHelper.sort(mNotificationList);

        if (notification.getSmallIcon() != null) {
            StatusBarNotification oldSbn = (old != null) ? old.sbn : null;
            mListeners.notifyPostedLocked(n, oldSbn);
        } else {
            Slog.e(TAG, "Not posting notification without small icon: " + notification);
            if (old != null && !old.isCanceled) {
                mListeners.notifyRemovedLocked(n);
            }
            // ATTENTION: in a future release we will bail out here
            // so that we do not play sounds, show lights, etc. for invalid
            // notifications
            Slog.e(TAG, "WARNING: In a future release this will crash the app: "
                    + n.getPackageName());
        }

        buzzBeepBlinkLocked(r);
    }
}

```

```java
void buzzBeepBlinkLocked(NotificationRecord record) {
    boolean buzz = false;
    boolean beep = false;
    boolean blink = false;

    final Notification notification = record.sbn.getNotification();
    final String key = record.getKey();

    // Should this notification make noise, vibe, or use the LED?
    final boolean aboveThreshold = record.getImportance() >= IMPORTANCE_DEFAULT;
    final boolean canInterrupt = aboveThreshold && !record.isIntercepted();
    if (DBG || record.isIntercepted())
        Slog.v(TAG,
                "pkg=" + record.sbn.getPackageName() + " canInterrupt=" + canInterrupt +
                        " intercept=" + record.isIntercepted()
        );

    final int currentUser;
    final long token = Binder.clearCallingIdentity();
    try {
        currentUser = ActivityManager.getCurrentUser();
    } finally {
        Binder.restoreCallingIdentity(token);
    }

    // If we're not supposed to beep, vibrate, etc. then don't.
    final String disableEffects = disableNotificationEffects(record);
    if (disableEffects != null) {
        ZenLog.traceDisableEffects(record, disableEffects);
    }

    // Remember if this notification already owns the notification channels.
    boolean wasBeep = key != null && key.equals(mSoundNotificationKey);
    boolean wasBuzz = key != null && key.equals(mVibrateNotificationKey);

    // These are set inside the conditional if the notification is allowed to make noise.
    boolean hasValidVibrate = false;
    boolean hasValidSound = false;
    if (disableEffects == null
            && (record.getUserId() == UserHandle.USER_ALL ||
                record.getUserId() == currentUser ||
                mUserProfiles.isCurrentProfile(record.getUserId()))
            && canInterrupt
            && mSystemReady
            && mAudioManager != null) {
        if (DBG) Slog.v(TAG, "Interrupting!");

        // should we use the default notification sound? (indicated either by
        // DEFAULT_SOUND or because notification.sound is pointing at
        // Settings.System.NOTIFICATION_SOUND)
        final boolean useDefaultSound =
               (notification.defaults & Notification.DEFAULT_SOUND) != 0 ||
                       Settings.System.DEFAULT_NOTIFICATION_URI
                               .equals(notification.sound);

        Uri soundUri = null;
        if (useDefaultSound) {
            soundUri = Settings.System.DEFAULT_NOTIFICATION_URI;

            // check to see if the default notification sound is silent
            ContentResolver resolver = getContext().getContentResolver();
            hasValidSound = Settings.System.getString(resolver,
                   Settings.System.NOTIFICATION_SOUND) != null;
        } else if (notification.sound != null) {
            soundUri = notification.sound;
            hasValidSound = (soundUri != null);
        }

        // Does the notification want to specify its own vibration?
        final boolean hasCustomVibrate = notification.vibrate != null;

        // new in 4.2: if there was supposed to be a sound and we're in vibrate
        // mode, and no other vibration is specified, we fall back to vibration
        final boolean convertSoundToVibration =
                !hasCustomVibrate
                        && hasValidSound
                        && (mAudioManager.getRingerModeInternal() == AudioManager.RINGER_MODE_VIBRATE);

        // The DEFAULT_VIBRATE flag trumps any custom vibration AND the fallback.
        final boolean useDefaultVibrate =
                (notification.defaults & Notification.DEFAULT_VIBRATE) != 0;

        hasValidVibrate = useDefaultVibrate || convertSoundToVibration ||
                hasCustomVibrate;

        // We can alert, and we're allowed to alert, but if the developer asked us to only do
        // it once, and we already have, then don't.
        if (!(record.isUpdate
                && (notification.flags & Notification.FLAG_ONLY_ALERT_ONCE) != 0)) {

            sendAccessibilityEvent(notification, record.sbn.getPackageName());

            if (hasValidSound) {
                boolean looping =
                        (notification.flags & Notification.FLAG_INSISTENT) != 0;
                AudioAttributes audioAttributes = audioAttributesForNotification(notification);
                mSoundNotificationKey = key;
                // do not play notifications if stream volume is 0 (typically because
                // ringer mode is silent) or if there is a user of exclusive audio focus
                if ((mAudioManager.getStreamVolume(
                        AudioAttributes.toLegacyStreamType(audioAttributes)) != 0)
                        && !mAudioManager.isAudioFocusExclusive()) {
                    final long identity = Binder.clearCallingIdentity();
                    try {
                        final IRingtonePlayer player =
                                mAudioManager.getRingtonePlayer();
                        if (player != null) {
                            if (DBG) Slog.v(TAG, "Playing sound " + soundUri
                                    + " with attributes " + audioAttributes);
                            player.playAsync(soundUri, record.sbn.getUser(), looping,
                                    audioAttributes);
                            beep = true;
                        }
                    } catch (RemoteException e) {
                    } finally {
                        Binder.restoreCallingIdentity(identity);
                    }
                }
            }

            if (hasValidVibrate && !(mAudioManager.getRingerModeInternal()
                    == AudioManager.RINGER_MODE_SILENT)) {
                mVibrateNotificationKey = key;

                if (useDefaultVibrate || convertSoundToVibration) {
                    // Escalate privileges so we can use the vibrator even if the
                    // notifying app does not have the VIBRATE permission.
                    long identity = Binder.clearCallingIdentity();
                    try {
                        mVibrator.vibrate(record.sbn.getUid(), record.sbn.getOpPkg(),
                                useDefaultVibrate ? mDefaultVibrationPattern
                                        : mFallbackVibrationPattern,
                                ((notification.flags & Notification.FLAG_INSISTENT) != 0)
                                        ? 0: -1, audioAttributesForNotification(notification));
                        buzz = true;
                    } finally {
                        Binder.restoreCallingIdentity(identity);
                    }
                } else if (notification.vibrate.length > 1) {
                    // If you want your own vibration pattern, you need the VIBRATE
                    // permission
                    mVibrator.vibrate(record.sbn.getUid(), record.sbn.getOpPkg(),
                            notification.vibrate,
                            ((notification.flags & Notification.FLAG_INSISTENT) != 0)
                                    ? 0: -1, audioAttributesForNotification(notification));
                    buzz = true;
                }
            }
        }

    }
    // If a notification is updated to remove the actively playing sound or vibrate,
    // cancel that feedback now
    if (wasBeep && !hasValidSound) {
        clearSoundLocked();
    }
    if (wasBuzz && !hasValidVibrate) {
        clearVibrateLocked();
    }

    // light
    // release the light
    boolean wasShowLights = mLights.remove(key);
    if ((notification.flags & Notification.FLAG_SHOW_LIGHTS) != 0 && aboveThreshold
            && ((record.getSuppressedVisualEffects()
            & NotificationListenerService.SUPPRESSED_EFFECT_SCREEN_OFF) == 0)) {
        mLights.add(key);
        updateLightsLocked();
        if (mUseAttentionLight) {
            mAttentionLight.pulse();
        }
        blink = true;
    } else if (wasShowLights) {
        updateLightsLocked();
    }
    if (buzz || beep || blink) {
        if (((record.getSuppressedVisualEffects()
                & NotificationListenerService.SUPPRESSED_EFFECT_SCREEN_OFF) != 0)) {
            if (DBG) Slog.v(TAG, "Suppressed SystemUI from triggering screen on");
        } else {
            EventLogTags.writeNotificationAlert(key,
                    buzz ? 1 : 0, beep ? 1 : 0, blink ? 1 : 0);
            mHandler.post(mBuzzBeepBlinked);
        }
    }
}


```


```java
void buzzBeepBlinkLocked(NotificationRecord record) 
```

```java
mHandler.post(mBuzzBeepBlinked)
```

```java
NotificationManagerService.java
private final Runnable mBuzzBeepBlinked = new Runnable() {
    @Override
    public void run() {
        if (mStatusBar != null) {
            mStatusBar.buzzBeepBlinked();
        }
    }
};
```

```java
StatusBarManagerInternal.java
void buzzBeepBlinked();
```

```java
StatusBarManagerService.java
public void buzzBeepBlinked() {
    if (mBar != null) {
        try {
            mBar.buzzBeepBlinked();
        } catch (RemoteException ex) {
        }
    }
}

```

负责通知灯闪烁的方法
```java
IStatusBar.aidl
void buzzBeepBlinked();

```


```java
CommandQueue.java
public void buzzBeepBlinked() 

```


```java
CommandQueue.java CallBacks
void buzzBeepBlinked();

```

```java
PhoneStatusBar.java
public void buzzBeepBlinked()
```

```java
PhoneStatusBar.java
public void fireBuzzBeepBlinked()
```


```java
DozeHost.java
void onBuzzBeepBlinked()

```


```java
DozeService.java
public void onBuzzBeepBlinked() {
    if (DEBUG) Log.d(mTag, "onBuzzBeepBlinked");
    updateNotificationPulse(System.currentTimeMillis());
}

```


```java
DozeService.java
private void updateNotificationPulse(long notificationTimeMs)
```


通知灯闪烁的地方, 不太确定?????
```java
DozeService.java
private void rescheduleNotificationPulse(boolean predicate) 
```

```java
AlarmManager.java
public void setExact(int type, long triggerAtMillis, PendingIntent operation)
```


```java
AlarmManager.java
private void setImpl(int type, long triggerAtMillis, long windowMillis, long intervalMillis,
        int flags, PendingIntent operation, final OnAlarmListener listener, String listenerTag,
        Handler targetHandler, WorkSource workSource, AlarmClockInfo alarmClock) 
```

```java
IAlarmManager.aidl
void set(String callingPackage, int type, long triggerAtTime, long windowLength,
            long interval, int flags, in PendingIntent operation, in IAlarmListener listener,
            String listenerTag, in WorkSource workSource, in AlarmManager.AlarmClockInfo alarmClock)
```



### 播放声音的地方


```java
if (hasValidSound) { //确定设置了声音, 如果是静音模式就不必播放呻吟了
    boolean looping =
            (notification.flags & Notification.FLAG_INSISTENT) != 0;
    AudioAttributes audioAttributes = audioAttributesForNotification(notification);
    mSoundNotificationKey = key;
    // do not play notifications if stream volume is 0 (typically because
    // ringer mode is silent) or if there is a user of exclusive audio focus
    if ((mAudioManager.getStreamVolume(
            AudioAttributes.toLegacyStreamType(audioAttributes)) != 0)
            && !mAudioManager.isAudioFocusExclusive()) {
        final long identity = Binder.clearCallingIdentity();
        try {
            final IRingtonePlayer player =
                    mAudioManager.getRingtonePlayer(); //获取RingtonePlayer对象
            if (player != null) {
                if (DBG) Slog.v(TAG, "Playing sound " + soundUri
                        + " with attributes " + audioAttributes);
                player.playAsync(soundUri, record.sbn.getUser(), looping,
                        audioAttributes); //转至SystemUI里的RingtonePlayer播放声音
                beep = true;
            }
        } catch (RemoteException e) {
        } finally {
            Binder.restoreCallingIdentity(identity);
        }
    }
}
```

```java
public void playAsync(Uri uri, UserHandle user, boolean looping, AudioAttributes aa) 

```


```java
public void play(Context context, Uri uri, boolean looping, AudioAttributes attributes)
```


```java
private void enqueueLocked(Command cmd)
```

```java
 public void run()
```


```java
private void startSound(Command cmd)
```


```java
CreationAndCompletionThread.java
public void run()

```

### 震动的地方

```java
if (hasValidVibrate && !(mAudioManager.getRingerModeInternal()
        == AudioManager.RINGER_MODE_SILENT)) {
    mVibrateNotificationKey = key;

    if (useDefaultVibrate || convertSoundToVibration) {
        // Escalate privileges so we can use the vibrator even if the
        // notifying app does not have the VIBRATE permission.
        long identity = Binder.clearCallingIdentity();
        try {
            mVibrator.vibrate(record.sbn.getUid(), record.sbn.getOpPkg(),
                    useDefaultVibrate ? mDefaultVibrationPattern
                            : mFallbackVibrationPattern,
                    ((notification.flags & Notification.FLAG_INSISTENT) != 0)
                            ? 0: -1, audioAttributesForNotification(notification));
            buzz = true;
        } finally {
            Binder.restoreCallingIdentity(identity);
        }
    } else if (notification.vibrate.length > 1) {
        // If you want your own vibration pattern, you need the VIBRATE
        // permission
        mVibrator.vibrate(record.sbn.getUid(), record.sbn.getOpPkg(),
                notification.vibrate,
                ((notification.flags & Notification.FLAG_INSISTENT) != 0)
                        ? 0: -1, audioAttributesForNotification(notification));
        buzz = true;
    }
}
```


```java
Vibrator.java
public abstract void vibrate(int uid, String opPkg, long[] pattern, int repeat,
        AudioAttributes attributes)

```


```java
SystemVibrator.java
public void vibrate(int uid, String opPkg, long[] pattern, int repeat,
        AudioAttributes attributes)

```

###  通知灯亮的地方
```java
NotificationManagerService.java
void updateLightsLocked()
```

```java
Light.java
public abstract void setFlashing(int color, int mode, int onMS, int offMS);
```

```java
LightsService.java
public void setFlashing(int color, int mode, int onMS, int offMS)
```


```java
LightsService.java
private void setLightLocked(int color, int mode, int onMS, int offMS, int brightnessMode)
```

```java
LightsService.java
static native void setLight_native(long ptr, int light, int color, int mode,
            int onMS, int offMS, int brightnessMode)
```

```java
StatusBarManagerInternal.java
void notificationLightPulse(int argb, int onMillis, int offMillis)
```


```java
StatusBarManagerService.java
public void notificationLightPulse(int argb, int onMillis, int offMillis)
```

```java
IStatusBar.aidl
void notificationLightPulse(int argb, int millisOn, int millisOff);
```

```java
CommandQueue.java
public void notificationLightPulse(int argb, int onMillis, int offMillis)
```


```java
CommandQueue.java
void notificationLightPulse(int argb, int onMillis, int offMillis);
```

```java
PhoneStatusBar.java
public void notificationLightPulse(int argb, int onMillis, int offMillis)
```

```java
PhoneStatusBar.java
public void fireNotificationLight(boolean on)
```

```java
DozeHost.java
void onNotificationLight(boolean on)
```


```java
DozeSerivce.java
public void onNotificationLight(boolean on)
```


```java
DozeSerivce.java
private void updateNotificationPulseDueToLight()
```

```java
DozeSerivce.java
private void updateNotificationPulse(long notificationTimeMs)
```


```java
DozeSerivce.java
private void rescheduleNotificationPulse(boolean predicate) {
```


```java
AlarmManager.java
 public void setExact(int type, long triggerAtMillis, PendingIntent operation) {
        setImpl(type, triggerAtMillis, WINDOW_EXACT, 0, 0, operation, null, null, null,
                null, null);
    }
```

### 通知的显示
```java
NotificationManagerService.java
EnqueueNotificationRunnable
public void run() 
```

```java
NotificationManagerService.java
public void notifyPostedLocked(StatusBarNotification sbn, StatusBarNotification oldSbn)
```

```java
NotificationManagerService.java
private void notifyPosted(final ManagedServiceInfo info,
                final StatusBarNotification sbn, NotificationRankingUpdate rankingUpdate)
```


```java
INotificationListener.aidl
void onNotificationPosted(in IStatusBarNotificationHolder notificationHolder,
        in NotificationRankingUpdate update
```


```java
NotificationListenerService.java
public void onNotificationPosted(IStatusBarNotificationHolder sbnHolder,
                NotificationRankingUpdate update)
```


```java
NotificationListenerService.java
public void handleMessage(Message msg);
```

```java
NotificationListenerService.java
public void onNotificationPosted(StatusBarNotification sbn, RankingMap rankingMap)
```

```java
NotificationListenerService.java
public void onNotificationPosted(StatusBarNotification sbn)
```

```java
BaseStatusBar.java
public void onNotificationPosted(final StatusBarNotification sbn,
                final RankingMap rankingMap)

```


```java
PhoneStatusBar.java
public void addNotification(StatusBarNotification notification, RankingMap ranking,
            Entry oldEntry) 
```


```java
PhoneStatusBar.java
protected void addNotificationViews(Entry entry, RankingMap ranking) 
//决定这通知是否下拉时在StatusBar上显示, 以及是否以弹出式的方式显示通知
```

```java
protected void updateNotifications()
```


```java
private void updateNotificationShade()
```


```java
StatusBarIconController.java
public void updateNotificationIcons(NotificationData notificationData)
```


```java
public void updateNotificationIcons(NotificationData notificationData)
//Updates the notifications with the given list of notifications to display.
```

```java
NotificationIconAreaController.java
private void applyNotificationIconsTint() 
```


```java
protected void setAreThereNotifications() //允许浮动式通知弹出
```


```java
PhoneStatusBar.java
public void findAndUpdateMediaNotifications()
```


```java
PhoneStatusBar.java
public void updateMediaMetaData(boolean metaDataChanged, boolean allowEnterAnimation) 
//更新或者是移除来自于媒体元数据的锁屏艺术封面或者锁屏壁纸
//Refresh or remove lockscreen artwork from media metadata or the lockscreen wallpaper.
```


### 通知的更新

```java
BaseStatusBar.java
boolean isUpdate = mNotificationData.get(key) != null  //通过这个条件来决定是否更新
```


```java
BaseStatusBar.java
public void updateNotification(StatusBarNotification notification, RankingMap ranking)
```


```java
BaseStatusBar.java
private void updateNotificationViews(Entry entry, StatusBarNotification sbn)
```


### 通知的取消
```java
NotificationManager.java
public void cancel(int id)
```

```java
NotificationManager.java
public void cancel(String tag, int id)
```


```java
NotificationManager.java
public void cancelAsUser(String tag, int id, UserHandle user)
```

```java
NotificationManagerService.java
public void cancelNotificationWithTag(String pkg, String tag, int id, int userId)
```

```java
NotificationManagerService.java
void cancelNotification(final int callingUid, final int callingPid,
            final String pkg, final String tag, final int id,
            final int mustHaveFlags, final int mustNotHaveFlags, final boolean sendDelete,
            final int userId, final int reason, final ManagedServiceInfo listener)
```


```java
NotificationManagerService.java
private void cancelNotificationLocked(NotificationRecord r, boolean sendDelete, int reason)
```


```java
PendingIntent.java
public void send() throws CanceledException
```


```java
PendingIntent.java
public void send(Context context, int code, @Nullable Intent intent,
            @Nullable OnFinished onFinished, @Nullable Handler handler,
            @Nullable String requiredPermission, @Nullable Bundle options)
            throws CanceledException

```


```java
public int sendIntentSender(IIntentSender target, int code, Intent intent, String resolvedType,
        IIntentReceiver finishedReceiver, String requiredPermission, Bundle options)
        throws RemoteException 
```

### Notification Duck



### NotificationUsageStats
```java
NotificationUsageStats.java
public synchronized void registerEnqueuedByApp(String packageName)
```


```java
NotificationUsageStats.java
private AggregatedStats[] getAggregatedStatsLocked(String packageName)
```

```java
NotificationUsageStats.java
private AggregatedStats getOrCreateAggregatedStatsLocked(String key)
```

```java
NotificationUsageStats.java
public AggregatedStats(Context context, String key)
```



### Notification点击跳转

```java
private final class NotificationClicker implements View.OnClickListener {
    public void onClick(final View v) {
        Log.d(TAG, "BaseStatusBar: NotificationClicker: onClick: ");
        if (!(v instanceof ExpandableNotificationRow)) {
            Log.e(TAG, "NotificationClicker called on a view that is not a notification row.");
            return;
        }

        final ExpandableNotificationRow row = (ExpandableNotificationRow) v;
        final StatusBarNotification sbn = row.getStatusBarNotification();
        if (sbn == null) {
            Log.e(TAG, "NotificationClicker called on an unclickable notification,");
            return;
        }

        // Check if the notification is displaying the gear, if so slide notification back
        if (row.getSettingsRow() != null && row.getSettingsRow().isVisible()) {
            row.animateTranslateNotification(0);
            return;
        }

        Notification notification = sbn.getNotification();
        final PendingIntent intent = notification.contentIntent != null
                ? notification.contentIntent
                : notification.fullScreenIntent;
        final String notificationKey = sbn.getKey();

        // Mark notification for one frame.
        row.setJustClicked(true);
        DejankUtils.postAfterTraversal(new Runnable() {
            @Override
            public void run() {
                row.setJustClicked(false);
            }
        });

        final boolean keyguardShowing = mStatusBarKeyguardViewManager.isShowing();
        final boolean afterKeyguardGone = intent.isActivity()
                && PreviewInflater.wouldLaunchResolverActivity(mContext, intent.getIntent(),
                        mCurrentUserId);
        dismissKeyguardThenExecute(new OnDismissAction() {
            public boolean onDismiss() {
                if (mHeadsUpManager != null && mHeadsUpManager.isHeadsUp(notificationKey)) {
                    // Release the HUN notification to the shade.

                    if (isPanelFullyCollapsed()) {
                        HeadsUpManager.setIsClickedNotification(row, true);
                    }
                    //
                    // In most cases, when FLAG_AUTO_CANCEL is set, the notification will
                    // become canceled shortly by NoMan, but we can't assume that.
                    mHeadsUpManager.releaseImmediately(notificationKey);
                }
                StatusBarNotification parentToCancel = null;
                if (shouldAutoCancel(sbn) && mGroupManager.isOnlyChildInGroup(sbn)) {
                    StatusBarNotification summarySbn = mGroupManager.getLogicalGroupSummary(sbn)
                                    .getStatusBarNotification();
                    if (shouldAutoCancel(summarySbn)) {
                        parentToCancel = summarySbn;
                    }
                }
                final StatusBarNotification parentToCancelFinal = parentToCancel;
                new Thread() {
                    @Override
                    public void run() {
                        try {
                            if (keyguardShowing && !afterKeyguardGone) {
                                ActivityManagerNative.getDefault()
                                        .keyguardWaitingForActivityDrawn();
                            }

                            // The intent we are sending is for the application, which
                            // won't have permission to immediately start an activity after
                            // the user switches to home.  We know it is safe to do at this
                            // point, so make sure new activity switches are now allowed.
                            ActivityManagerNative.getDefault().resumeAppSwitches();
                        } catch (RemoteException e) {
                        }
                        if (intent != null) {
                            // If we are launching a work activity and require to launch
                            // separate work challenge, we defer the activity action and cancel
                            // notification until work challenge is unlocked.
                            if (intent.isActivity()) {
                                final int userId = intent.getCreatorUserHandle()
                                        .getIdentifier();
                                if (mLockPatternUtils.isSeparateProfileChallengeEnabled(userId)
                                        && mKeyguardManager.isDeviceLocked(userId)) {
                                    if (startWorkChallengeIfNecessary(userId,
                                            intent.getIntentSender(), notificationKey)) {
                                        // Show work challenge, do not run pendingintent and
                                        // remove notification
                                        return;
                                    }
                                }
                            }
                            try {
                                intent.send(null, 0, null, null, null, null,
                                        getActivityOptions());
                            } catch (PendingIntent.CanceledException e) {
                                // the stack trace isn't very helpful here.
                                // Just log the exception message.
                                Log.w(TAG, "Sending contentIntent failed: " + e);

                                // TODO: Dismiss Keyguard.
                            }
                            if (intent.isActivity()) {
                                mAssistManager.hideAssist();
                                overrideActivityPendingAppTransition(keyguardShowing
                                        && !afterKeyguardGone);
                            }
                        }

                        try {
                            mBarService.onNotificationClick(notificationKey);
                        } catch (RemoteException ex) {
                            // system process is dead if we're here.
                        }
                        if (parentToCancelFinal != null) {
                            // We have to post it to the UI thread for synchronization
                            mHandler.post(new Runnable() {
                                @Override
                                public void run() {
                                    Runnable removeRunnable = new Runnable() {
                                        @Override
                                        public void run() {
                                            performRemoveNotification(parentToCancelFinal,
                                                    true);
                                        }
                                    };
                                    if (isCollapsing()) {
                                        // To avoid lags we're only performing the remove
                                        // after the shade was collapsed
                                        addPostCollapseAction(removeRunnable);
                                    } else {
                                        removeRunnable.run();
                                    }
                                }
                            });
                        }
                    }
                }.start();

                // close the shade if it was open
                animateCollapsePanels(CommandQueue.FLAG_EXCLUDE_RECENTS_PANEL,
                        true /* force */, true /* delayed */);
                visibilityChanged(false);

                return true;
            }
        }, afterKeyguardGone);
    }

    private boolean shouldAutoCancel(StatusBarNotification sbn) {
        int flags = sbn.getNotification().flags;
        if ((flags & Notification.FLAG_AUTO_CANCEL) != Notification.FLAG_AUTO_CANCEL) {
            return false;
        }
        if ((flags & Notification.FLAG_FOREGROUND_SERVICE) != 0) {
            return false;
        }
        return true;
    }

    public void register(ExpandableNotificationRow row, StatusBarNotification sbn) {
        Notification notification = sbn.getNotification();
        if (notification.contentIntent != null || notification.fullScreenIntent != null) {
            row.setOnClickListener(this);
        } else {
            row.setOnClickListener(null);
        }
    }
}

```


### Notification没有小图标
```java
public void notifyRemovedLocked(StatusBarNotification sbn)
```

```java
private void notifyRemoved(ManagedServiceInfo info, StatusBarNotification sbn,
        NotificationRankingUpdate rankingUpdate)
```


### Record思考
```
am/ActivityRecord.java
am/AppBindRecord.java
am/BackupRecord.java
am/BroadcastRecord.java
am/ConnectionRecord.java
am/ContentProviderRecord.java
am/IntentBindRecord.java
am/PendingIntentRecord.java
am/ProcessRecord.java
am/ServiceRecord.java
am/TaskRecord.java
UidRecord.java
media/MediaSessionRecord.java
notification/NotificationRecord.java
```


## 从架构上了解Notification


```
1. 查看notification经过了几个进程
2. 各个进程之间使用了哪些AIDL 来通讯
3. 通讯之间使用了哪些类作为参数
4. 

```

Notification经过的进程
```
1. CallerPID: 调用了Notificationmanager 的notify方法, 这个仍属于调用的所在的进程, 跨进程使用了INotificationManager.aidl
2. system_server: NotificationManagerService 所在的进程是system_server, 在这里进行了逻辑的处理, 跨进程采用了INotificationListener.aidl
3. com.android.systemui: SystemUI进行界面处理. 是com.android.systemui 进程
4. com.android.systemui: SystemUI 播放铃声, 是com.android.systemui 进程. 跨进程使用了IRingtonePlayer.aidl
5. com.android.systemui: SystemUI Notification灯亮, 是com.android.systemui 进程. 跨进程使用了IStatusBar.aidl
6. 

```



涉及到的AIDL, 因为AIDL是进程间的通信,一次可能涉及到多个进程
```
IStatusBar.aidl
IStatusBarNotificationHolder.aidl
IStatusBarService.aidl
Notification.aidl
INotificationManager
IRingtonePlayer.aidl
INotificationListener.aidl
IStatusBarService.aidl
IStatusBarNotificationHolder.aidl

```


### 相关问题
```
1. 通知是如何发出去的
2. 通知是如何播放铃声的
3. 发通知的方式
4. 发完通知没看通知, 通知灯的闪烁
5. android的framework生成的几个进程, system_process, com.android.systemui, android.ext.services, android.process_acore, android.process.media
6. 通知是如何更新的
7. 通知是如何显示的
8. 同一个应用的多个通知是如何被组织到一起的
9. Notification的架构
10. StatusBarIconView的显示, 被包围在IconMerger里面
11. 通知更新的条件是什么
12. 通知以组的形式显示的条件是什么
13. 通知的种类
14. Notification Duck
15. NotificationRecord的作用
16. NotificationListenerService的作用
17. 认识Notification
18. PendingIntent是如何添加button的
19. Notification的样式

```
```
NotificationRecord  在NotificationManagerService里用此类来记录Notification
Notification  
Ranking 存储着当前正在活跃的notification的等级信息
NotificationListenerService  
NotificationRankerService 继承于NotificationListenerService

```
涉及到的模块
```
SystemUI
services.core

```


```java
mNotificationData.add(entry, ranking);  //注释掉来通知的时候， 出现两个StatusBar， 下拉没有通知
updateNotifications();  //注释没有下拉通知， 重启SystemUI通知会出来
```




```java
final ArrayList<NotificationRecord> mNotificationList =
    new ArrayList<NotificationRecord>();
final ArrayMap<String, NotificationRecord> mNotificationsByKey =
    new ArrayMap<String, NotificationRecord>();
final ArrayMap<Integer, ArrayMap<String, String>> mAutobundledSummaries = new ArrayMap<>();
final ArrayList<ToastRecord> mToastQueue = new ArrayList<ToastRecord>();
final ArrayMap<String, NotificationRecord> mSummaryByGroupKey = new ArrayMap<>();
final PolicyAccess mPolicyAccess = new PolicyAccess();

// The last key in this list owns the hardware.
ArrayList<String> mLights = new ArrayList<>();


```


### 锁屏通知布局

```
StatusBarWindowView  //整个下拉通知,包括状态栏, 锁屏通知
NotificationPanelView //包含所有的显示的通知, 包含状态栏
NotificationsQuickSettingsContainer//包含所有的显示的通知, 包含状态栏, 不包含底部的锁屏KeyGuradBottom
NotificationStackScrollLayout //只包含所有的通知ExpandableNotificationRow以及过多的通知显示栏OverflowContainer,
ExpandableNotificationRow
NotificationBackgroundView// 一个normal状态， 一个dimmed状态
NotificationContentView


```
文档有待整理

## 参考文献

1. [Notification](https://developer.android.com/reference/android/app/Notification.html)
2. [Notifications, Android 4.4 and Lower](https://developer.android.com/design/patterns/notifications_k.html)
3. [Android 4.4 KitKat NotificationManagerService使用详解与原理分析(一)__使用详解](http://blog.csdn.net/yihongyuelan/article/details/40977323)
4. [ Android 4.4 KitKat NotificationManagerService使用详解与原理分析(二)__原理分析](http://blog.csdn.net/yihongyuelan/article/details/41084165)
5. [Android N 通知概览及example](http://www.jianshu.com/p/d9fbcb0db013)
6. [从开发者角度解析 Android N 新特性！](https://gank.io/post/56e0b83c67765963436fcb94)
7. [译探寻 Android N 中通知的新变化](https://gold.xitu.io/entry/577a27e76be3ff006a1ef870)
8. [Android5.1应用统计源码分析](http://blog.csdn.net/elder_sword/article/details/50809642)
9. [ Android 4.0 ICS SystemUI浅析——StatusBar加载流程之Notification](http://blog.csdn.net/yihongyuelan/article/details/7751013)
10. [ Android 通知栏Notification的整合 全面学习 ](http://blog.csdn.net/vipzjyno1/article/details/25248021)
11. [Improved notifications with Direct reply in Android N](https://segunfamisa.com/posts/notifications-direct-reply-android-nougat)
11. [你真的了解Android Notification吗?](http://connorlin.github.io/2016/04/21/%E4%BD%A0%E7%9C%9F%E7%9A%84%E4%BA%86%E8%A7%A3Android-Notification%E5%90%97/)
12. [Android 开发之 Notification 详解](http://glgjing.github.io/blog/2015/11/18/android-kai-fa-zhi-notification-xiang-jie/)
13. [Android基础——Notification](http://yuweiguocn.github.io/android-basic-notification/)
14. [Android N新特性：direct reply notification](http://mahong978.top/2016/09/08/Android-DRN/)
15. [Android Notification 通知样式总结](https://shoewann0402.github.io/2016/05/12/Android-Notification-%E9%80%9A%E7%9F%A5%E6%A0%B7%E5%BC%8F%E6%80%BB%E7%BB%93/)
16. [android notification完全解析](http://canglangwenyue.com/2014/12/08/android-notification%E5%AE%8C%E5%85%A8%E8%A7%A3%E6%9E%90/)
17. [Android的Notification](http://haoxiqiang.github.io/blog/20150129-AndroidNotification.html)
18. [Android笔记之Notification](http://lovenight.github.io/2015/12/10/Android%E7%AC%94%E8%AE%B0%E4%B9%8BNotification/)
19. [Notification通知](http://ghvn7777.github.io/2016/05/27/Notification/)
20. [android通知、PendingIntent、定时机制](http://wn398.github.io/2015/03/16/android/android%E9%80%9A%E7%9F%A5%E3%80%81PendingIntent%E3%80%81%E5%AE%9A%E6%97%B6%E6%9C%BA%E5%88%B6/)
21. [Android学习日记21--消息提示之Toast和Notification](http://chendd.com/blog/2013/03/25/android_study_21/  )
22. [Android的StatusBar分析](http://blog.csdn.net/jdsjlzx/article/details/6916533)
23. [Android SystemUI 分析——通知](http://light3moon.com/2015/02/06/Android%20SystemUI%20%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E9%80%9A%E7%9F%A5/)
24. [Notification数据结构和功能处理流程分析](http://blog.csdn.net/jamikabin/article/details/22950735)
25. [ Android NotificationListenerService原理简介](http://blog.csdn.net/yueqian_scut/article/details/51417547)
26. [什么是 Dirty hack](https://www.zhihu.com/question/20372589)
27. [android 特殊用户通知用法汇总--Notification源码分析](http://blog.csdn.net/self_study/article/details/51055769)
28. [Notification的显示过程](http://blog.csdn.net/thinkingword/article/details/7769621)
29. [Notification与NotificationManager详细 _Android](https://yq.aliyun.com/ziliao/98493)
30. [ Android5.x Notification应用解析 ](http://blog.csdn.net/itachi85/article/details/50096609)
31. [Notification机制浅析（基于SDK23）](http://www.jianshu.com/p/3ed68162e1f5)
32. [ NotificationManagerService笔记](http://blog.csdn.net/guoqifa29/article/details/44133905)
33. [Android基础入门教程——2.5.2 Notification(状态栏通知)详解](https://www.zybuluo.com/coder-pig/note/183026)
34. [Android 7.0 SystemUI 之启动和状态栏和导航栏简介](http://blog.csdn.net/qq_31530015/article/details/53507968)
35. [Notification framework层的处理流程分析](http://fortianwei.iteye.com/blog/1180020)
36. [Android通知栏介绍与适配总结](http://iluhcm.com/2017/03/12/experience-of-adapting-to-android-notifications/)
[]()
[]()
[]()
[]()
[]()
[]()
[]()
[]()
[]()
[]()
