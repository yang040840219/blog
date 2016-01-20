---
layout: post
title: spark broadcast
tags:  [broadcast]
categories: [broadcast]
author: mingtian
excerpt: "spark"
---


### Spark 广播


相关类

* BroadcastManager
* BroadcastFactory  子类 HttpBroadcastFactory, TorrentBroadcastFactory
* HttpBroadcastFactory
* TorrentBroadcastFactory (默认)
* Broadcast 子类 TorrentBroadcast, HttpBroadcast
* TorrentBroadcast
* HttpBroadcast


> 创建

SparkContext

~~~
  // 创建 BroadcastManager
  val broadcastManager = new BroadcastManager(isDriver, conf, securityManager)
  
  // 创建 Broadcast
  val bc = env.broadcastManager.newBroadcast[T](value, isLocal)
~~~

~~~
   private def writeBlocks(value: T): Int = {
    // Store a copy of the broadcast variable in the driver so that tasks run on the driver
    // do not create a duplicate copy of the broadcast variable's value.
    SparkEnv.get.blockManager.putSingle(broadcastId, value, StorageLevel.MEMORY_AND_DISK,
      tellMaster = false)
    val blocks =
      TorrentBroadcast.blockifyObject(value, blockSize, SparkEnv.get.serializer, compressionCodec)
    blocks.zipWithIndex.foreach { case (block, i) =>
      SparkEnv.get.blockManager.putBytes(
        BroadcastBlockId(id, "piece" + i),
        block,
        StorageLevel.MEMORY_AND_DISK_SER,
        tellMaster = true)
    }
    blocks.length
  }
~~~

创建 TorrentBroadcast  生成 Broadcast id 通过 writeBlocks 方法 把 value 值 分成 多个 block 保存到 BlockManager 中， block的大小可以通过 spark.broadcast.blockSize 指定
  
 
> 读取

~~~
 bc.value(xx)  // 具体调用 getValue 方法
~~~

TorrentBroadcast

~~~
 SparkEnv.get.blockManager.getLocal(broadcastId).map(_.data.next()) 
~~~

从本地获取


~~~
 val blocks = readBlocks()
  val obj = TorrentBroadcast.unBlockifyObject[T](
            blocks, SparkEnv.get.serializer, compressionCodec)
~~~

本地获取不到后 ， 会调用 readBlocks , 按照 block 获取 Broadcast ，可以从 本地或者从远程 ， 最后把 blocks 转换成  value 
