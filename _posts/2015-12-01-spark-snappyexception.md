---
layout: post
title: Spark run unit snappy exception
tags:  [Spark]
categories: [Spark]
author: mingtian
excerpt: "Spark run unit snappy exception"
---



~~~
java.lang.reflect.InvocationTargetException
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:606)
    at org.xerial.snappy.SnappyLoader.loadNativeLibrary(SnappyLoader.java:317)
    at org.xerial.snappy.SnappyLoader.load(SnappyLoader.java:219)
    at org.xerial.snappy.Snappy.<clinit>(Snappy.java:44)
    at org.apache.spark.io.SnappyCompressionCodec.<init>(CompressionCodec.scala:150)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:57)
    at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
    at java.lang.reflect.Constructor.newInstance(Constructor.java:526)
    at org.apache.spark.io.CompressionCodec$.createCodec(CompressionCodec.scala:68)
    at org.apache.spark.io.CompressionCodec$.createCodec(CompressionCodec.scala:60)
    at org.apache.spark.broadcast.TorrentBroadcast.org$apache$spark$broadcast$TorrentBroadcast$$setConf(TorrentBroadcast.scala:73)
    at org.apache.spark.broadcast.TorrentBroadcast.<init>(TorrentBroadcast.scala:79)
    at org.apache.spark.broadcast.TorrentBroadcastFactory.newBroadcast(TorrentBroadcastFactory.scala:34)
    at org.apache.spark.broadcast.BroadcastManager.newBroadcast(BroadcastManager.scala:62)
    at org.apache.spark.SparkContext.broadcast(SparkContext.scala:1077)
    at org.apache.spark.scheduler.DAGScheduler.org$apache$spark$scheduler$DAGScheduler$$submitMissingTasks(DAGScheduler.scala:849)
    at org.apache.spark.scheduler.DAGScheduler.org$apache$spark$scheduler$DAGScheduler$$submitStage(DAGScheduler.scala:790)
    at org.apache.spark.scheduler.DAGScheduler$$anonfun$org$apache$spark$scheduler$DAGScheduler$$submitStage$4.apply(DAGScheduler.scala:793)
    at org.apache.spark.scheduler.DAGScheduler$$anonfun$org$apache$spark$scheduler$DAGScheduler$$submitStage$4.apply(DAGScheduler.scala:792)
    at scala.collection.immutable.List.foreach(List.scala:318)
    at org.apache.spark.scheduler.DAGScheduler.org$apache$spark$scheduler$DAGScheduler$$submitStage(DAGScheduler.scala:792)
    at org.apache.spark.scheduler.DAGScheduler$$anonfun$org$apache$spark$scheduler$DAGScheduler$$submitStage$4.apply(DAGScheduler.scala:793)
    at org.apache.spark.scheduler.DAGScheduler$$anonfun$org$apache$spark$scheduler$DAGScheduler$$submitStage$4.apply(DAGScheduler.scala:792)
    at scala.collection.immutable.List.foreach(List.scala:318)
    at org.apache.spark.scheduler.DAGScheduler.org$apache$spark$scheduler$DAGScheduler$$submitStage(DAGScheduler.scala:792)
    at org.apache.spark.scheduler.DAGScheduler.handleJobSubmitted(DAGScheduler.scala:774)
    at org.apache.spark.scheduler.DAGSchedulerEventProcessLoop.onReceive(DAGScheduler.scala:1393)
    at org.apache.spark.scheduler.DAGSchedulerEventProcessLoop.onReceive(DAGScheduler.scala:1385)
    at org.apache.spark.util.EventLoop$$anon$1.run(EventLoop.scala:48)
Caused by: java.lang.UnsatisfiedLinkError: no snappyjava in java.library.path
    at java.lang.ClassLoader.loadLibrary(ClassLoader.java:1878)
    at java.lang.Runtime.loadLibrary0(Runtime.java:849)
    at java.lang.System.loadLibrary(System.java:1087)
    at org.xerial.snappy.SnappyNativeLoader.loadLibrary(SnappyNativeLoader.java:52)
    ... 33 more
~~~

 解决: idea -> run -> edit configurations -> vm params

~~~
-Dorg.xerial.snappy.lib.name=libsnappyjava.jnilib -Dorg.xerial.snappy.tempdir=/tmp
~~~ 
