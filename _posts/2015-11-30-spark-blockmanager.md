---
layout: post
title: Spark BlockManager
tags:  [Spark]
categories: [Spark]
author: mingtian
excerpt: "Spark BlockManager"
---

## Spark BlockManager

### 相关类

* BlockManagerId 对应一个BlockManager,可能运行在Driver端，或者是Executor 端 new BlockManagerId(execId, host, port)

* BlockManagerInfo  用来对应 BlockManagerId 和 创建的BlockManager对应的RpcEndpoint的 ref ，有个updateBlockInfo方法，在BlockManagerMasterEndpoint 端 用来更新各个 Block 的信息(BlockInfo)

* BlockId 定义一个block 表示一个文件, 子类  RDDBlockId ，ShuffleBlockId，TaskResultBlockId(Task计算完成结果超过设置的值，先保存在本地) 保存或者读取时的标识

* BlockManagerMasterEndpoint 持有所有Executor中BlockManager的ref（通过BlockManagerInfo封装，定义Map类型blockManagerInfo 保存），block操作的方法，具体的逻辑实现（RegisterBlockManager，UpdateBlockInfo）

* BlockManagerSlaveEndpoint 和 BlockManagerMasterEndpoint 通信,在Executor端执行Driver 的命令

* BlockManagerMaster 封装了对Block的操作，调用BlockManagerMasterEndpoint执行

* BlockInfo  Block 的 Storage Level

* BlockManager  Block 存储和读取的调用接口

* BlockResult 用来包装根据BlockId 从 MemoryStore/DiskStore 获取的数据结果

* ManagedBuffer  子类  FileSegmentManagedBuffer, NioManagedBuffer, NettyManagedBuffer

* BlockStore   子类 DiskStore, ExternalBlockStore, MemoryStore

                       

### 创建方式

BlockManager 分别在 Driver 和 Executor中在创建SparkEnv时创建

* 在Driver端创建

~~~
   // Create the Spark execution environment (cache, map output tracker, etc)
    _env = createSparkEnv(_conf, isLocal, listenerBus)
    SparkEnv.set(_env)
    
 val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(
      BlockManagerMaster.DRIVER_ENDPOINT_NAME,
      new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),
      conf, isDriver)

    // NB: blockManager is not valid until initialize() is called later.
  val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
      serializer, conf, mapOutputTracker, shuffleManager, blockTransferService, securityManager,
      numUsableCores)
~~~ 
1. 创建 BlockManagerMaster 
	registerOrLookupEndpoint方法如果在driver端执行，通过创建的new BlockManagerMasterEndpoint 的对象放到 RpcEnv 中返回ref,在Executor中，直接返回ref
2. BlockManagerMaster 在创建时有 BlockManagerMasterEndpoint 后续的操作大部分通过此EndPoint完成
3. 创建BlockManager
	 参数executorId 为 driver
4. 调用 initialize 方法  _env.blockManager.initialize(_applicationId)
	 
* 在Executor端创建  

~~~
     val env = SparkEnv.createExecutorEnv(
        driverConf, executorId, hostname, port, cores, isLocal = false)
~~~
1. 参数 isLocal = false, executorId 对应具体的Executor， 
2. 对应创建SparkEnv 时 isDriver=false, BlockManagerMaster 中包含的是BlockManagerMasterEndpoint的ref
3. 在Executor.scala 中 env.blockManager.initialize(conf.getAppId)

### BlockManager 初始化

~~~
private[spark] class BlockManager(
    executorId: String,
    rpcEnv: RpcEnv,
    val master: BlockManagerMaster,
    defaultSerializer: Serializer,
    maxMemory: Long,
    val conf: SparkConf,
    mapOutputTracker: MapOutputTracker,
    shuffleManager: ShuffleManager,
    blockTransferService: BlockTransferService,
    securityManager: SecurityManager,
    numUsableCores: Int)
  extends BlockDataManager with Logging
~~~

MapOutputTracker: 跟踪一个Stage的map 输出位置  
ShuffleManager: shuffle 系统的提供的接口， SortShuffleManager 、HashShuffleManager  
BlockTransferService: 用来传输Block数据，NettyBlockTransferService、 NioBlockTransferService  


~~~
  def initialize(appId: String): Unit = {
    blockTransferService.init(this)
    shuffleClient.init(appId)

    blockManagerId = BlockManagerId(
      executorId, blockTransferService.hostName, blockTransferService.port)

    shuffleServerId = if (externalShuffleServiceEnabled) {
      BlockManagerId(executorId, blockTransferService.hostName, externalShuffleServicePort)
    } else {
      blockManagerId
    }

    master.registerBlockManager(blockManagerId, maxMemory, slaveEndpoint)

    // Register Executors' configuration with the local shuffle service, if one should exist.
    if (externalShuffleServiceEnabled && !blockManagerId.isDriver) {
      registerWithExternalShuffleServer()
    }
  }
~~~

初始化 ShuffleClient:ExternalShuffleClient , BlockTransferService , master
为 BlockManagerMaster, registerBlockManager 方法把创建的BlockManagerSlaveEndpoint注册到 
BlockManagerMasterEndpoint上,事件名称为 RegisterBlockManager, BlockManagerMasterEndpoint接收到事件后,调用register(blockManagerId, maxMemSize, slaveEndpoint) 方法, 创建和 salveEndpoint 对应的 BlockManagerInfo


### 调用过程

> http://jerryshao.me/architecture/2013/10/08/spark-storage-module-analysis/

1. 写入过程

    考虑ShuffleMapTask 
    
~~~
      val manager = SparkEnv.get.shuffleManager
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
~~~
    
    writer 为 SortShuffleWriter 
    
~~~
      val partitionLengths = sorter.writePartitionedFile(blockId, context, outputFile)
      shuffleBlockResolver.writeIndexFile(dep.shuffleId, mapId, partitionLengths)
~~~
   
    shuffleBlockResolver 为 IndexShuffleBlockResolver  调用 ExternalSorter 中的 writePartitionedFile 
    
~~~
      val writer = blockManager.getDiskWriter(blockId, outputFile, serInstance, fileBufferSize,
            context.taskMetrics.shuffleWriteMetrics.get)
~~~
    
    调用 blockManager 中的 方法写入数据，blockId 标识写入的数据
    
    另外是直接调用 BlockManager 中的 putBytes、putArray 方法 在 BlockManager 内部统一使用 doPut 方法
    
~~~
     private def doPut(
      blockId: BlockId,
      data: BlockValues,
      level: StorageLevel,
      tellMaster: Boolean = true,
      effectiveStorageLevel: Option[StorageLevel] = None)
    : Seq[(BlockId, BlockStatus)]
~~~
    
    
2. 读取过程
   



