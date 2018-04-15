---
title: log4j2配置详解
date: 2017-12-11 14:48:21
tags: log4j2
---

# 输出规则：

<console>标签是控制台，表示在控制台进行输出。

Log4j由三个重要的组成的构建组成：记录器(Loggers)、输出源（Appenders）、

和布局(Layouts)

 

记录器的Level是日志记录的优先级：debug<info<warn<error

记录器的level级别为info，那么这个记录器记录的级别就是>= info,即info、warn、error。

 

# 输出源：

org.apache.log4j.ConsoleAppender(控制台)

org.apache.log4j.FileAppender(文件)

org.apache.log4j.DailyRollingFileAppender(每天产生一个日志文件)

org.apache.log4j.RollingFileAppender(文件大小到达指定尺寸的时候产生一个新的文件)

org.apache.log4j.WriterAppender(将日志信息以流的格式发送到任意指定的地方)

org.apache.log4j.SocketAppender(socket)

org.apache.log4j.NtEventLogAppender(NT的Event Log)

org.apache.log4j.JMSAppender(电子邮件)

 

# layout(布局)

## 布局种类
org.apache.log4j.HTMLLayout（以HTML表格形式布局）

org.apache.log4j.PatternLayout(可以灵活地指定布局模式)

org.apache.log4j.SimpleLayout(包含日志信息的级别和信息字符串)

org.apache.log4j.TTCCLayout(包含日志产生的时间、线程、类别等等信息)

 

如果使用了PatternLayout，则Log4j采用类似C语言中println函数的打印格式格式化日志信息

%m 输出代码中的指定消息

%P 输出优先级，即debug、info、warn、error、fatal

%r 输出自应用启动到输出该log信息耗费的毫秒数

%c 输出所属的类目，通常就是所在类的全名

%t 输出产生该日志事件的线程名

%n 输出一个回车换行符,Windows平台下为”/n/r”，Unix平台为“/n”

%d 输出日志时间点的日期或事件，默认格式为ISO8601，也可以在其后制定格式。

%l 输出日志事件的发生位置，包括类目名、发生的线程，以及在代码中的行数。

 

Configuration标签

Status属性：有八种，可以检测log4j2的配置文件是否有错，也可以检测到死循环的logger。

monitorInterval：自动检测配置文件的事件间隔，单位是秒，最小间隔是5秒。Log4j2检测到配置文件有变化，会重新配置自己。

## 输出源标签：

<console>是将日志输出到控制台上。

<RollingFile>

Filename:日志存放路径

bufferedIO：是否有缓存IO

bufferSize：bufferSize内存大小

filePattern：压缩后的路径

PatternLayout下的pattern：日志输出的内容

SizeBasedTriggeringPolicy：容量达到多少进行压缩

Loggers：

Additivity：false 避免重复打印

 

log4j2.xml样本
``` xml

<?xml version="1.0" encoding="UTF-8"?>

 

<Configuration status="OFF" >

 

<Appenders>

<Console name="console" target="SYSTEM_OUT">

<PatternLayout pattern="[%d{yyyy-MM-dd HH:mm:ss,SSS}][%p][%c{1}][Line:%L] %m%n" />

</Console>

 

<RollingFile name="clearcontrolLog" fileName="/amots/appfiles/logs/clearcontrol/clearcontrolLog.log" bufferedIO="true"

bufferSize="8192" filePattern="/amots/appfiles/logs/clearcontrol/clearcontrolLog-%d{yyyy-MM-dd-HH}-%i.log.gz">

<PatternLayout pattern="[%d{yyyy-MM-dd HH:mm:ss,SSS}][%p][%c{1}][Line:%L] %m%n" />

<Policies>

<TimeBasedTriggeringPolicy interval="24" />

<SizeBasedTriggeringPolicy size="30MB" />

</Policies>

</RollingFile>

 

<RollingFile name="clearofbaseLog" fileName="/amots/appfiles/logs/clearofbase/clearofbaseLog.log" bufferedIO="true"

bufferSize="8192" filePattern="/amots/appfiles/logs/clearofbase/clearofbaseLog-%d{yyyy-MM-dd-HH}-%i.log.gz">

<PatternLayout pattern="[%d{yyyy-MM-dd HH:mm:ss,SSS}][%p][%c{1}][Line:%L] %m%n" />

<Policies>

<TimeBasedTriggeringPolicy interval="24" />

<SizeBasedTriggeringPolicy size="30MB" />

</Policies>

</RollingFile>

</Appenders>

 

<Loggers>

 

<!-- amots -->

<Logger name="clearcontrol" level="INFO" additivity="false">

<!-- <AppenderRef ref="console" /> -->

<AppenderRef ref="clearcontrolLog" />

</Logger>

<Logger name="clearofbase" level="INFO" additivity="false">

<!-- <AppenderRef ref="console" /> -->

<AppenderRef ref="clearcontrolLog" />

</Logger>

</Loggers>

</Configuration>

```
 

注：

1、应用中不可直接使用日志系统(log4j、Logback)中的api，而应依赖使用日志框架的SLF4J中的框架，使用门面模式的日志模式的框架、有利于维护和各个类的日志处理方式统一。

``` java
import org.slf4j.Logger;

import org.slf4j.LoggerFactory;

private static final Logger logger=LoggerFactory.getLogger(ABC.class);

```

2、对trace/debug/ingo级别的日志输出，必须使用条件形式或者使用占位符的方式。在使用logger输出时，如果配置级别为info，那么在输出日志时，如果不加，日志不会打印，但是会执行字符串拼接。对象会执行toString()方法，浪费系统资源，执行了操作，还没有打印日志。

应加
``` java

if(logger.isDebugEnacled()){

    logger.debug("");

}

```
或者
```java

logger.debug("processing debug with id:{} symbol:{}",id,symbol)

```