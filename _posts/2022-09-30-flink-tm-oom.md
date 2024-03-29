---
layout: post
title: Flink TaskManager OOM 问题排查
author: 细雪
header-style: text
lang: en
published: true
categories: Flink
tags:
  - Flink
---

## 打印GC日志
可以通过加入如下JVM参数，发生OOM时进行堆栈打印，通常是由于内存泄漏问题导致的OOM

-Xloggc:/opt/flink/log/gc.log GC 日志的存储位置
-XX:PrintGCApplicationStoppedTime 输出 GC 时应用暂停时间
-XX:+PrintGCDetails 输出 GC 的详细日志
+PrintGCDateStamps 输出 GC 的日期时间，格式为 2021-07-28T18:38:44.743+0800
-XX:+UseGCLogFileRotation 开启日志文件分割
-XX:NumberOfGCLogFiles:NumberOfGCLogFiles 最多分割几个 GC 日志文件
-XX:GCLogFileSize=10M 每个日志文件上限大小，超过就会触发分割
-XX:+PrintPromotionFailure 输出导致 GC 的原因
-XX:+HeapDumpOnOutOfMemoryError 发生内存泄漏时，自动打印 heapdump 文件
-XX:HeapDumpPath=/opt/flink/log/heap.hprof heapdump 文件路径

## 排查GC时间
-XX:+UseG1GC -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=128M -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCApplicationConcurrentTime

## 排查GC次数
```
jstat -gcutil {pid}  1000 100
说明：
-gc
     堆内存回收
     S0C: 当前幸存者区0的容量 (kB).
     S1C: 当前幸存者区1的容量(kB).
     S0U: 幸存者区0已用内存 (kB).
     S1U: 幸存者区1已用内存 (kB).
     EC: 伊甸园区容量 (kB).
     EU: 伊甸园区已用内存 (kB).
     OC: 当前老旧区容量 (kB).
     OU: 老旧区已用内存 (kB).
     MC: 元数据区容量 (kB).
     MU: 元数据区已用内存 (kB).
     CCSC: 类压缩区容量 (kB).
     CCSU: 类压缩区已用内存 (kB).
     YGC: 新生垃圾回收事件数量.
     YGCT: 新生垃圾回收时间.
     FGC: 垃圾回收事件总和.
     FGCT: 完整的一次垃圾回收时间.
     GCT: 所有的垃圾回收时间.
```