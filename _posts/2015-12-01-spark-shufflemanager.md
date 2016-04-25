---
layout: post
title: Spark ShuffleManager
tags:  [Spark]
categories: [Spark]
author: mingtian
excerpt: "Spark ShuffleManager"
---


## ShuffleManager

>  定义 ShuffleWriter 和 ShuffleReader 具体的实现类,在Driver 和 Executor 端都会创建，具体的写入和读取调用 ShuffleWriter 和 ShuffleReader实现

* HashShuffleManager
* SortShuffleManager
* UnsafeShuffleManager

``` spark
   // 在SparkEnv中
    val shortShuffleMgrNames = Map(
      "hash" -> "org.apache.spark.shuffle.hash.HashShuffleManager",
      "sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager",
      "tungsten-sort" -> "org.apache.spark.shuffle.unsafe.UnsafeShuffleManager")
    val shuffleMgrName = conf.get("spark.shuffle.manager", "sort") // 默认使用 SortShuffleManager
    val shuffleMgrClass = shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase, shuffleMgrName)
    val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)
```

## ShuffleReader

> 读取 shuffle 后的各个Executor 本地的数据

* HashShuffleReader

## ShuffleWriter

> 写入shuffle 数据到各个Executor的本地

* HashShuffleWriter
* SortShuffleWriter (默认)
* UnsafeShuffleWriter

## ShuffleBlockResolver

 > ShuffleBlockResolver 的实现类，需要实现 block data 和  shuffle block 的 对应关系，在BlockStore中调用。

 * IndexShuffleBlockResolver

 > 创建和维护Shuffle 中 逻辑上的Block 和 物理文件的Block 之间的对应关系， 一个Map中的数据，会保存到一个数据文件中，idnex file 保存数据的offset
 > 使用 shuffleBlockId 和 reduce Id 命名文件
 
 * FileShuffleBlockResolver

## ShuffleHandle
>  保存 Shuffle 的信息, ShuffleId 、numMaps 、 ShuffleDependency

* BaseShuffleHandle
* UnsafeShuffleHandle
 
 


##ShuffleMapTask

ShuffleMapTask 的 rdd 可以是 MapPartitionsRDD, ShuffleManager 会把rdd中每个Partition的数据写入到Exectuor的本地

 ``` spark
      val manager = SparkEnv.get.shuffleManager
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
      return writer.stop(success = true).get
 ```
 	
``` spark
     override def getWriter[K, V](handle: ShuffleHandle, mapId: Int, context: TaskContext)
      : ShuffleWriter[K, V] = {
    val baseShuffleHandle = handle.asInstanceOf[BaseShuffleHandle[K, V, _]]
    shuffleMapNumber.putIfAbsent(baseShuffleHandle.shuffleId, baseShuffleHandle.numMaps)
    new SortShuffleWriter(
      shuffleBlockResolver, baseShuffleHandle, mapId, context)
  }
```

SparkEnv 默认的 ShuffleManager 是  org.apache.spark.shuffle.sort.SortShuffleManager 对应的 getWriter 方法会创建 SortShuffleWriter 主要的参数是 ShuffleHandle , partitionId 对应后续的 mapId。
在 SortShuffleWriter 的 write 方法中调用 ExternalSorter 的 insertAll 方法
  
```
    val outputFile = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
    val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
    val partitionLengths = sorter.writePartitionedFile(blockId, context, outputFile)
    shuffleBlockResolver.writeIndexFile(dep.shuffleId, mapId, partitionLengths)
```

shuffleBlockResolver 为 IndexShuffleBlockResolver 数据写入到 SPARK_LOCAL_DIRS/XXX

> eg: /Users/yxl/data/spark.dir/spark-1d3ddd71-0bf8-498e-bef9-514f9aaa7bbf/blockmgr-cafc764d-92f2-4fa3-bc2a-c0facfb4be03/38/shuffle_0_4_0.data


## ResultTask

ResultTask 的 rdd 可以是 ShuffledRDD , 在 执行 runTask 方法时，会直接调用

~~~
  func(context, rdd.iterator(partition, context))
~~~

创建 ShuffledRDD ，DAGScheduler 划分 stage时会调用 ShuffledRDD 的  getDependencies 创建 ShuffleMapStage

``` spark
   new ShuffledRDD[K, V, C](self, partitioner)
        .setSerializer(serializer)
        .setAggregator(aggregator)
        .setMapSideCombine(mapSideCombine)
```

ShuffledRDD 的 compute() 根据 rdd 的 ShuffleDependency 创建 ShuffleReader

``` spark
override def compute(split: Partition, context: TaskContext): Iterator[(K, C)] = {
    val dep = dependencies.head.asInstanceOf[ShuffleDependency[K, V, C]]
    SparkEnv.get.shuffleManager.getReader(dep.shuffleHandle, split.index, split.index + 1, context)
      .read()
      .asInstanceOf[Iterator[(K, C)]]
  }
```


![PartitionedAppendOnlyMap继承关系](/blog/assets/images/post/map.jpeg.001.jpeg)




