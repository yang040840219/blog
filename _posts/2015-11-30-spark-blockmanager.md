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

* BlockManagerInfo  用来对应 BlockManagerId 和 创建的BlockManager对应的RpcEndpoint的 ref ，有个updateBlockInfo方法

* BlockId 定义一个block, 子类  RDDBlockId ，ShuffleBlockId，TaskResultBlockId

* BlockManagerMasterEndpoint 持有所有Executor中BlockManager的ref（通过BlockManagerInfo封装，定义Map类型blockManagerInfo 保存），block操作的方法，具体的逻辑实现（RegisterBlockManager，UpdateBlockInfo）

* BlockManagerSlaveEndpoint 和 BlockManagerMasterEndpoint 通信,在Executor端执行Driver 的命令

* BlockManagerMaster 封装了对Block的操作，调用BlockManagerMasterEndpoint执行


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
	 
* 在Executor端创建  

~~~
     val env = SparkEnv.createExecutorEnv(
        driverConf, executorId, hostname, port, cores, isLocal = false)
~~~
1. 参数 isLocal = false, executorId 对应具体的Executor， 
2. 对应创建SparkEnv 时 isDriver=false, BlockManagerMaster 中包含的是BlockManagerMasterEndpoint的ref

### BlockManager 创建过程

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



	 

	
