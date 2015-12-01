---
layout: post
title: Spark ShuffleManager
tags:  [spark]
categories: [spark]
author: mingtian
excerpt: "Spark ShuffleManager"
---


## Spark ShuffleManager

* HashShuffleManager
* SortShuffleManager


## ShuffleBlockResolver

 > ShuffleBlockResolver 的实现类，需要实现 block data 和  shuffle block 的 对应关系，在BlockStore中调用。

 * IndexShuffleBlockResolver
 > 创建和维护Shuffle 中 逻辑上的Block 和 物理文件的Block 之间的对应关系， 一个Map中的数据，会保存到一个数据文件中，idnex file 保存数据的offset
 > 使用 shuffleBlockId 和 reduce Id 命名文件
 
 * FileShuffleBlockResolver
 


###ShuffleMapTask

~~~
      val manager = SparkEnv.get.shuffleManager
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
      return writer.stop(success = true).get
~~~
~~~
     override def getWriter[K, V](handle: ShuffleHandle, mapId: Int, context: TaskContext)
      : ShuffleWriter[K, V] = {
    val baseShuffleHandle = handle.asInstanceOf[BaseShuffleHandle[K, V, _]]
    shuffleMapNumber.putIfAbsent(baseShuffleHandle.shuffleId, baseShuffleHandle.numMaps)
    new SortShuffleWriter(
      shuffleBlockResolver, baseShuffleHandle, mapId, context)
  }
~~~

SparkEnv 默认的 ShuffleManager 是  org.apache.spark.shuffle.sort.SortShuffleManager 对应的 getWriter 方法会创建 SortShuffleWriter 主要的参数是 ShuffleHandle , partitionId 对应后续的 mapId。
在 write 方法中调用 ExternalSorter 的 insertAll 方法
  
~~~
    val outputFile = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
    val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
    val partitionLengths = sorter.writePartitionedFile(blockId, context, outputFile)
    shuffleBlockResolver.writeIndexFile(dep.shuffleId, mapId, partitionLengths)
~~~
shuffleBlockResolver 为 IndexShuffleBlockResolver 数据写入到 SPARK_LOCAL_DIRS/XXX
> eg: /Users/yxl/data/spark.dir/spark-1d3ddd71-0bf8-498e-bef9-514f9aaa7bbf/blockmgr-cafc764d-92f2-4fa3-bc2a-c0facfb4be03/38/shuffle_0_4_0.data


### ResultTask



