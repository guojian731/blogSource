---
title: JVM-2-常用命令
date: 2017-12-11 17:48:41
tags: JVM
---
# 堆内存设定
-Xmx ：最大堆内存

-Xms ：初始堆内存 

# 占空间设定
-Xss ：指定线程最大栈空间

# 方法区：
**理解为永久区perm，在jdk1.6和1.7中存在，在1.8中剔除**

-XX:PermSize：初始化方法区。

-XX:MaxPermSize:最大方法区：默认情况为64M

# 输出

-XX:+PrintGC 打印gc日志

-XX:+PrintGCDetails  打印详细的gc日志

-XX:+PrintHeapAtGC 在每次GC前后分别打印堆的信息。

-XX:+PrintGCTimeStamps 该参数会在每个GC发生时，额外的输出GC发生的时间，该输出时间为虚拟机启动后的时间偏移量。

-XX:+PrintGCApplicationConcurrentTime 打印应用程序的执行时间

-XX:+PrintGCApplicationStoppedTime 打印应用程序由于GC而产生的停顿时间。

-XX:+PrintReferenceGC 跟踪系统中软引用、弱引用、虚引用和Finallize队列。

-Xloggc:log/gc.log  在当前目录下的log文件夹下的log文件夹下的gc.log文件中记录所有的GC日志。

# 类加载/卸载的跟踪

-verbose:class 跟踪类的加载和卸载

-XX:+TranceClassLoading 跟踪类的加载

-XX:+TranceClassUnloading 跟踪类的卸载

Java虚拟机还允许研发人员在运行时打印、查看系统中类的分布情况。只要在系统中启动时加上-XX:+PrintClassHistogram参数，然后在Java的控制台中按下Ctrl+Break组合键，控制台上就会显示当前的类信息柱状图。

 

# 系统参数查看：

-XX:+PrintVMOptions 在程序运行时，打印虚拟机接收到的命令行显示参数。

-XX:+PrintCommandLineFlags 可以打印传递给虚拟机的显示和隐示参数。

-XX:+PrintFlagsFinal 打印所有的系统参数的值（大约有500多个）

# 新生代配置：

-Xmn 用于设置新生代的大小。新生代大小一般设置为整个堆空间的1/3到1/4之间。(-Xmn1m)

-XX:SurvivorRatio=eden/from=eden/to 设置新生代中eden空间和from/to空间的比例关系。

（-XX:SurvivorRatio=2 新生代内存有10m，那么eden区的内存即为5m）

-XX:NewRatio=老年代/新生代 设置新生代和老年代的比例（与-Xmn区别是一个是设置新生代绝对大小，一个是根据比例设置）

# 堆溢出处理：

-XX:+HeapDumpOnOutOfMemoryError 在内存溢出时到处整个堆信息。

-XX:+HeapDumpPath:d:/a.dmp 和-XX:+HeapDumpOnOutOfMemoryError配合使用，导出堆的存放路径。
