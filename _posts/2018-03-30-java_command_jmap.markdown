---
layout:     post
title:      "Java 命令 （一）"
subtitle:   "Java 命令 （一）"
date:       2018-03-30 00:10:40
author:     "Mathew"
catalog: true
header-img: "img/post-bg-2017.jpg"
tags:
    - Android
    - Notification
---

---
layout:     post
title:      "Java 命令 （一）"
subtitle:   "Java 命令 （一）"
date:       2017-06-25 00:10:40
author:     "Mathew"
catalog: true
header-img: "img/post-bg-2017.jpg"
tags:
    - Android
    - Notification
---

# Java 命令 （一）
本命令的java环境是
```
openjdk version "1.8.0_161"
OpenJDK Runtime Environment (build 1.8.0_161-b14)
OpenJDK 64-Bit Server VM (build 25.161-b14, mixed mode)
```
第一句是 jdk的版本信息
第二句是jre的运行环境信息
第三句是虚拟机的版本信息
mixed mode 是JVM的工作模式。 
我们可以在命令行使用 java -X 命令获取java的所有的工作模式.
以下是JVM的所有的工作模式， mixed mode是默认的工作模式
```
    -Xmixed           mixed mode execution (default)
    -Xint             interpreted mode execution only
    -Xbootclasspath:<directories and zip/jar files separated by :>
                      set search path for bootstrap classes and resources
    -Xbootclasspath/a:<directories and zip/jar files separated by :>
                      append to end of bootstrap class path
    -Xbootclasspath/p:<directories and zip/jar files separated by :>
                      prepend in front of bootstrap class path
    -Xdiag            show additional diagnostic messages
    -Xnoclassgc       disable class garbage collection
    -Xincgc           enable incremental garbage collection
    -Xloggc:<file>    log GC status to a file with time stamps
    -Xbatch           disable background compilation
    -Xms<size>        set initial Java heap size
    -Xmx<size>        set maximum Java heap size
    -Xss<size>        set java thread stack size
    -Xprof            output cpu profiling data
    -Xfuture          enable strictest checks, anticipating future default
    -Xrs              reduce use of OS signals by Java/VM (see documentation)
    -Xcheck:jni       perform additional checks for JNI functions
    -Xshare:off       do not attempt to use shared class data
    -Xshare:auto      use shared class data if possible (default)
    -Xshare:on        require using shared class data, otherwise fail.
    -XshowSettings    show all settings and continue
    -XshowSettings:all
                      show all settings and continue
    -XshowSettings:vm show all vm related settings and continue
    -XshowSettings:properties
                      show all property settings and continue
    -XshowSettings:locale
                      show all locale related settings and continue

The -X options are non-standard and subject to change without notice.
```
## Jmap命令
Jmap 是JVM Memory Map for Java的意思。
### Jmap Error
#### VMVersionMismatchException
执行java命令的jmap命令, 这个时候你会发现。VMVersionMismatchException 异常。
适因为, jmap在的jdk的版本和当前正在运行的java程序的虚拟集版本号不一致导致的. 所以我们得尽量保持版本号的一直
```
[xuwanjin@xuwanjin Downloads]$ jmap -heap 30343
Attaching to process ID 30343, please wait...
Error attaching to process: sun.jvm.hotspot.runtime.VMVersionMismatchException: Supported versions are 25.161-b14. Target VM is 25.152-b01
sun.jvm.hotspot.debugger.DebuggerException: sun.jvm.hotspot.runtime.VMVersionMismatchException: Supported versions are 25.161-b14. Target VM is 25.152-b01
	at sun.jvm.hotspot.HotSpotAgent.setupVM(HotSpotAgent.java:435)
	at sun.jvm.hotspot.HotSpotAgent.go(HotSpotAgent.java:305)
	at sun.jvm.hotspot.HotSpotAgent.attach(HotSpotAgent.java:140)
	at sun.jvm.hotspot.tools.Tool.start(Tool.java:185)
	at sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
	at sun.jvm.hotspot.tools.HeapSummary.main(HeapSummary.java:49)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at sun.tools.jmap.JMap.runTool(JMap.java:201)
	at sun.tools.jmap.JMap.main(JMap.java:130)
Caused by: sun.jvm.hotspot.runtime.VMVersionMismatchException: Supported versions are 25.161-b14. Target VM is 25.152-b01
	at sun.jvm.hotspot.runtime.VM.checkVMVersion(VM.java:227)
	at sun.jvm.hotspot.runtime.VM.<init>(VM.java:294)
	at sun.jvm.hotspot.runtime.VM.initialize(VM.java:370)
	at sun.jvm.hotspot.HotSpotAgent.setupVM(HotSpotAgent.java:431)
	... 11 more

```
#### DebuggerException
有时后你会发现你使用jmap 命令的时候， 出现如下的
```
[xuwanjin@xuwanjin Downloads]$ jmap -heap 24863
Attaching to process ID 24863, please wait...
Error attaching to process: sun.jvm.hotspot.debugger.DebuggerException: cannot open binary file
sun.jvm.hotspot.debugger.DebuggerException: sun.jvm.hotspot.debugger.DebuggerException: cannot open binary file
	at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$LinuxDebuggerLocalWorkerThread.execute(LinuxDebuggerLocal.java:163)
	at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.attach(LinuxDebuggerLocal.java:278)
	at sun.jvm.hotspot.HotSpotAgent.attachDebugger(HotSpotAgent.java:671)
	at sun.jvm.hotspot.HotSpotAgent.setupDebuggerLinux(HotSpotAgent.java:611)
	at sun.jvm.hotspot.HotSpotAgent.setupDebugger(HotSpotAgent.java:337)
	at sun.jvm.hotspot.HotSpotAgent.go(HotSpotAgent.java:304)
	at sun.jvm.hotspot.HotSpotAgent.attach(HotSpotAgent.java:140)
	at sun.jvm.hotspot.tools.Tool.start(Tool.java:185)
	at sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
	at sun.jvm.hotspot.tools.HeapSummary.main(HeapSummary.java:49)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at sun.tools.jmap.JMap.runTool(JMap.java:201)
	at sun.tools.jmap.JMap.main(JMap.java:130)
Caused by: sun.jvm.hotspot.debugger.DebuggerException: cannot open binary file
	at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.attach0(Native Method)
	at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.access$100(LinuxDebuggerLocal.java:62)
	at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$1AttachTask.doit(LinuxDebuggerLocal.java:269)
	at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$LinuxDebuggerLocalWorkerThread.run(LinuxDebuggerLocal.java:138)
```
#### InvocationTargetException
```
[xuwanjin@xuwanjin Downloads]$ jmap -heap 1905
Attaching to process ID 1905, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.161-b14

using thread-local object allocation.
Parallel GC with 8 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 4148166656 (3956.0MB)
   NewSize                  = 86507520 (82.5MB)
   MaxNewSize               = 1382547456 (1318.5MB)
   OldSize                  = 173539328 (165.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at sun.tools.jmap.JMap.runTool(JMap.java:201)
	at sun.tools.jmap.JMap.main(JMap.java:130)
Caused by: java.lang.RuntimeException: unknown CollectedHeap type : class sun.jvm.hotspot.gc_interface.CollectedHeap
	at sun.jvm.hotspot.tools.HeapSummary.run(HeapSummary.java:144)
	at sun.jvm.hotspot.tools.Tool.startInternal(Tool.java:260)
	at sun.jvm.hotspot.tools.Tool.start(Tool.java:223)
	at sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
	at sun.jvm.hotspot.tools.HeapSummary.main(HeapSummary.java:49)
	... 6 more

```
在执行jmap的时候出现了运行时异常 unknown CollectedHeap type : class 。
遇到这类问题的第一反应是google了一下。 原来需要下载openjdk的debug 包
如果你需要下载的是1.7的， 那么直接把java-1.8.0-openjdk-debuginfo 改成java-1.7.0-openjdk-debuginfo即可
```
sudo yum --enablerepo="*-debug*" install java-1.8.0-openjdk-debuginfo
```

#### Jmap
安装了相对应的debuginfo的包之后在此执行jmap -heap 1905。 当然你的获取的结果可能和我们不一样。
但是基本上都是相似的
```
[xuwanjin@xuwanjin Downloads]$ jmap -heap 1905
Attaching to process ID 1905, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.161-b14

using thread-local object allocation.
Parallel GC with 8 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 4148166656 (3956.0MB)
   NewSize                  = 86507520 (82.5MB)
   MaxNewSize               = 1382547456 (1318.5MB)
   OldSize                  = 173539328 (165.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 460849152 (439.5MB)
   used     = 2403528 (2.2921829223632812MB)
   free     = 458445624 (437.2078170776367MB)
   0.5215433270451152% used
From Space:
   capacity = 460849152 (439.5MB)
   used     = 0 (0.0MB)
   free     = 460849152 (439.5MB)
   0.0% used
To Space:
   capacity = 460849152 (439.5MB)
   used     = 0 (0.0MB)
   free     = 460849152 (439.5MB)
   0.0% used
PS Old Generation
   capacity = 2272788480 (2167.5MB)
   used     = 310437944 (296.0566940307617MB)
   free     = 1962350536 (1871.4433059692383MB)
   13.658901685386931% used

122748 interned Strings occupying 12973760 bytes.
```

对于jdk是1.7.0_171的
```
[xuwanjin@xuwanjin android-6.0.0_r62]$ jmap -heap 10505
Attaching to process ID 10505, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.171-b01

using thread-local object allocation.
Parallel GC with 8 thread(s)

Heap Configuration:
   MinHeapFreeRatio = 0
   MaxHeapFreeRatio = 100
   MaxHeapSize      = 4148166656 (3956.0MB)
   NewSize          = 1310720 (1.25MB)
   MaxNewSize       = 17592186044415 MB
   OldSize          = 5439488 (5.1875MB)
   NewRatio         = 2
   SurvivorRatio    = 8
   PermSize         = 21757952 (20.75MB)
   MaxPermSize      = 174063616 (166.0MB)
   G1HeapRegionSize = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 1097334784 (1046.5MB)
   used     = 511275992 (487.59078216552734MB)
   free     = 586058792 (558.9092178344727MB)
   46.592525768325594% used
From Space:
   capacity = 128450560 (122.5MB)
   used     = 43382016 (41.372314453125MB)
   free     = 85068544 (81.127685546875MB)
   33.773317920918366% used
To Space:
   capacity = 134742016 (128.5MB)
   used     = 0 (0.0MB)
   free     = 134742016 (128.5MB)
   0.0% used
PS Old Generation
   capacity = 1789919232 (1707.0MB)
   used     = 613412392 (584.9956436157227MB)
   free     = 1176506840 (1122.0043563842773MB)
   34.27039505657426% used
PS Perm Generation
   capacity = 38797312 (37.0MB)
   used     = 19204984 (18.31529998779297MB)
   free     = 19592328 (18.68470001220703MB)
   49.500810777818835% used

51340 interned Strings occupying 4654976 bytes.

```

同样在jdk 8 的版本下，你也可能遇到不同的结果
```
[xuwanjin@xuwanjin bin]$ jmap -heap 7454
Attaching to process ID 7454, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.161-b14

using parallel threads in the new generation.
using thread-local object allocation.
Concurrent Mark-Sweep GC

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 268435456 (256.0MB)
   MaxNewSize               = 697892864 (665.5625MB)
   OldSize                  = 6291456 (6.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 241631232 (230.4375MB)
   used     = 156614080 (149.35882568359375MB)
   free     = 85017152 (81.07867431640625MB)
   64.8153298328587% used
Eden Space:
   capacity = 214827008 (204.875MB)
   used     = 129809856 (123.79632568359375MB)
   free     = 85017152 (81.07867431640625MB)
   60.42529624580537% used
From Space:
   capacity = 26804224 (25.5625MB)
   used     = 26804224 (25.5625MB)
   free     = 0 (0.0MB)
   100.0% used
To Space:
   capacity = 26804224 (25.5625MB)
   used     = 0 (0.0MB)
   free     = 26804224 (25.5625MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 292990976 (279.41796875MB)
   used     = 175793440 (167.64968872070312MB)
   free     = 117197536 (111.76828002929688MB)
   59.999608998196585% used

22581 interned Strings occupying 1975680 bytes.
```

而最新的java 9 已经一出了-heap 这个参数了