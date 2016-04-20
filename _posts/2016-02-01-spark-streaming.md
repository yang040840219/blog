---
layout: post
title: Spark Streaming
tags:  [Spark]
categories: [Spark]
author: mingtian
excerpt: "Spark Streaming"
---


spark.version : 1.4


### Spark Streaming

```
  val ssc = new StreamingContext(sparkConf, Seconds(2))
  ssc.checkpoint("checkpoint")
  val lines = KafkaUtils.createStream(ssc, zkQuorum, group, topicMap).map(_._2)
  lines.print()
  ssc.start()
  ssc.awaitTermination()
```

![Spark Streaming 运行](/blog/assets/images/post/spark-streaming/spark-streaming.003.jpeg)


#### StreamingContext

1. 创建 DStreamGraph 如果有 Checkpoint 则从 Checkpoint 目录启动
2. 创建 JobScheduler
3. 创建 ContextWaiter
4. 创建 StreamingSource

调用 start 方法后 主要是执行 JobScheduler 的 start 方法


#### ReceiverInputDStream

 所有继承该类的 InputDStream 都是要有对应的Receiver 实现，后续会运行在 Work 上的。
 定义 从 Blocks 转换为 BlockRDD 的 compute 方法


#### DStreamGraph

```
val newGraph = new DStreamGraph()
newGraph.setBatchDuration(batchDur_) // 程序中设置的Seconds(10)
newGraph
```
StreamingContext 初始化 DStreamGraph 时 会传递一个 batchDuration。
DStreamGraph 持有所有的 InputStream 和 OutputStream 

InputStream 是在代码, 中创建 InputDStream的子类时在父类调用 ssc.graph.addInputStream(this) 时 添加的。 同样每个 InputDStream 也会持有 DStreamGraph 的引用 

```
 def addInputStream(inputStream: InputDStream[_]) {
    this.synchronized {
      inputStream.setGraph(this)  // 具体的执行在DStream 中
      inputStreams += inputStream
    }
  }

```


```
  val lines = ssc.textFileStream(directory)
  
  // 最后创建一个 KafkaInputDStream 在 父类中添加到 DStreamGraph 中
  val lines = KafkaUtils.createStream(ssc, zkQuorum, group, topicMap).map(_._2)
```

OutputStream 是在有action 操作时产生的 

```
 new ForEachDStream(this, context.sparkContext.clean(foreachFunc)).register()
				|
				|

// DStream
 private[streaming] def register(): DStream[T] = {
    ssc.graph.addOutputStream(this)
    this
  }
  
```

foreachRDD 说明： 

```
def foreachRDD(foreachFunc: (RDD[T], Time) => Unit): Unit = ssc.withScope {
    // because the DStream is reachable from the outer object here, and because
    // DStreams can't be serialized with closures, we can't proactively check
    // it for serializability and so we pass the optional false to SparkContext.clean
    new ForEachDStream(this, context.sparkContext.clean(foreachFunc, false)).register()
  }

```

最终创建 ForEachDStream 注册到 DStreamGraph 的 OutputStream 上。 注意用法就好！

```
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

#### JobScheduler

初始化时会接收一个 StreamingContext 对象。

1. 创建 JobGenerator 
2. 创建 ReceiverTracker 
3. 创建 InputInfoTracker 


#### ReceiverTracker

![ReceiverTracker](/blog/assets/images/post/spark-streaming/spark-streaming.008.jpeg)

过滤添加到 DStreamGraph 中 继承ReceiverInputDStream类的所有 InputDStream 如 KafkaInputDStream, 创建用来启动所有Receiver(每个InputStream 都有自己的Receiver)的 ReceiverLauncher 类,其 start 方法中 receiverExecutor.start() , ReceiverExecutor 会在单独
的线程中执行 startReceivers() 。 同时创建 ReceiverTrackerEndpoint 同在各个Worker 上创建的Receiver 进行通信。创建ReceiverTrackerEndpoint 用来用来管理所有的Receiver 用 ReceiverInfo 类 来表示所有注册到 ReceiverTracker 中的 Receiver 、接收Receiver发送的数据

![ReliableKafkaReceiver](/blog/assets/images/post/spark-streaming/spark-streaming.009.jpeg)

![ReceiverSupervisor](/blog/assets/images/post/spark-streaming/spark-streaming.010.jpeg)


从 ReceiverTracker 的 start() 开始

```
 if (!receiverInputStreams.isEmpty) {
      endpoint = ssc.env.rpcEnv.setupEndpoint(
        "ReceiverTracker", new ReceiverTrackerEndpoint(ssc.env.rpcEnv))
      if (!skipReceiverLaunch) receiverExecutor.start()
      logInfo("ReceiverTracker started")
    }
```

启动 ReceiverTrackerEndpoint 用来同后续启动的Receiver 通信 ，处理 RegisterReceiver、AddBlock、DeregisterReceiver 等事件
ReceiverLauncher 会在新线程中调用 startReceivers() , 把 InputStreams 封装成RDD 在 work 上执行，启动 ReceiverSupervisor
调用 ReceiverSupervisor 的 start 方法

```
     // 在work的Executor上运行的 Receiver 的具体函数，最后调用 ReceiverSupervisorImpl 的 start 方法
      val startReceiver = (iterator: Iterator[Receiver[_]]) => {
        if (!iterator.hasNext) {
          throw new SparkException(
            "Could not start receiver as object not found.")
        }
        val receiver = iterator.next()
        val supervisor = new ReceiverSupervisorImpl(
          receiver, SparkEnv.get, serializableHadoopConf.value, checkpointDirOption)
        supervisor.start()
        supervisor.awaitTermination()
      }
```

			Receiver
				|
		KafkaReceiver
		
Receiver 中 保存数据, 启动, 停止 都是调用 ReceiverSupervisor 中的方法。 在ReceiverSupervisor 中 receiver.attachExecutor(this), 让Receiver 持有  ReceiverSupervisor 的引用。
		
		ReceiverSupervisor
				|
	 ReceiverSupervisorImpl


ReceiverSupervisor 的 start 方法 会调用自身的onStart() 实现类 ReceiverSupervisorImpl在 onStart() 中实例化 BlockGenerator 类, 然后调用 startReceiver() 继而调用 具体 Receiver(InputDStream 对应的Receiver (KafaInputDStream 对应的是KafkaReceiver 通过 getReceiver() 创建)) 的 onStart() , 最后调用 onReceiverStart 把 启动的Receiver 注册到 ReceiverTrackerEndpoint 上。
Receiver 启动后开始收集数据,调用 store() 方法保存收集到的数据

```
ReceiverSupervisorImpl 
      |
      |		             ReceiverSupervisorImpl
      |  ReceiverSupervisor      |
      |			|               |                      
    start()----onStart() ---onStart() // 创建BlockGenerator                        
    								|                              
    								|             
   ReceiverSupervisor -- startReceiver() 
    		                    |
    		                    |             							 ------------------			              |                 |
    		        receiver.onStart()   onReceiverStart()  
    		             |				       |
    		             |                    |      
    			   KafkaReceiver		ReceiverSupervisorImpl 
```

保存数据流程

```

 Receiver.store(bytes: ByteBuffer)
      |
      |
 ReceiverSupervisorImpl.pushBytes
  	   |
  	   |
 ReceiverSupervisorImpl.pushAndReportBlock 	   |
 	   |
 //  具体的执行保存数据逻辑 BlockManagerBasedBlockHandler
 //  和 WriteAheadLogBasedBlockHandler 两个实现类
 ReceivedBlockHandler.storeBlock  
 	   |
 	   |
 // 发送消息给 ReceiverTrakcer 
 trackerEndpoint.askWithRetry[Boolean](AddBlock(blockInfo))  	   |
 	   |
 ReceiverTracker.addBlock
  	   |
  	   |
 ReceivedBlockTracker.addBlock	
 
```


#### ReceivedBlockTracker

在 ReceiverTracker 中创建，用来跟踪接收到的Blocks, 然后根据 jobScheduler.receiverTracker.allocateBlocksToBatch(time) 的调用 把接收到的block分成一批，内部操作都是基于 WAL 的


#### JobGenerator

![JobGenerator](/blog/assets/images/post/spark-streaming/spark-streaming.004.jpeg)

调用EventLoop 定时的生成 job(GenerateJobs),通过 processEvent 来处理。同时还有 ClearMetadata、DoCheckpoint、ClearCheckpointData等事件。
生成job 

```
private def processEvent(event: JobGeneratorEvent) {
    logDebug("Got event " + event)
    event match {
      case GenerateJobs(time) => generateJobs(time)
      case ClearMetadata(time) => clearMetadata(time)
      case DoCheckpoint(time, clearCheckpointDataLater) =>
        doCheckpoint(time, clearCheckpointDataLater)
      case ClearCheckpointData(time) => clearCheckpointData(time)
    }
  }

```

主要考虑  generateJobs(time) 方法

```
private def generateJobs(time: Time) {
    // Set the SparkEnv in this thread, so that job generation code can access the environment
    // Example: BlockRDDs are created in this thread, and it needs to access BlockManager
    // Update: This is probably redundant after threadlocal stuff in SparkEnv has been removed.
    SparkEnv.set(ssc.env)
    Try {
      jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batch
      graph.generateJobs(time) // generate jobs using allocated block
    } match {
      case Success(jobs) =>
        val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)
        val streamIdToNumRecords = streamIdToInputInfos.mapValues(_.numRecords)
        jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToNumRecords))
      case Failure(e) =>
        jobScheduler.reportError("Error generating jobs for time " + time, e)
    }
    eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = false))
  }
```

1.调用 ReceiverTracker 中的 allocateBlockToBatch 方法，把当前所有的Stream Id 对应的 ReceiveBlockInfo 信息汇总到一起，封装成 AllocatedBlocks 返回（具体的实现是 ReceiverBlockTracker 中的allocateBlocksToBatch()），并且写日志。 数据保存在 timeToAllocatedBlocks:HashMap 中，每个Stream Id 对应 batchtime 内的 ReceiverBlockInfo 可以通过  ReceiverTracker 中的 getBlocksOfBatch(time) 和 getBlocksOfBatchAndStream(time,stream) 获取

```
 writeToLog(BatchAllocationEvent(batchTime, allocatedBlocks)) 
```


2.graph.generateJobs(time) 针对每个注册的 OutputStream 执行 DStream 的子类会重写这个方法，比如 ForEachDStream 生成一个job

```
 override def generateJob(time: Time): Option[Job] = {
    parent.getOrCompute(time) match {
      case Some(rdd) =>
        val jobFunc = () => createRDDWithLocalProperties(time) {
          ssc.sparkContext.setCallSite(creationSite)
          foreachFunc(rdd, time)
        }
        Some(new Job(time, jobFunc))
      case None => None
    }
  }
  
```

```
 val lines = ssc.textFileStream("/Users/yxl/data/spark-streaming")
 val words = lines.flatMap(_.split(" "))
 val wordCounts = words.map(x => (x, 1)).reduceByKey(_ + _)
 wordCounts.print()
```

根据以上代码 生成的 DStream 继承关系

```
FileInputDStream
	FlatMappedDStream
		MappedDStream
			ShuffledDStream
				ForEachDStream
```
parent.getOrCompute(time) 调用 DStream 的getOrCompute 根据 time 返回 RDD , 最后调用到 FileInputDStream 的 compute 方法，FileInputDStream 中的compute 方法实现比较简单，找到目录下新增的文件，把每个文件转成RDD 最后合并到一起。 如果是继承自ReceiverInputDStream 的 InputStream 会调用 ReceiverInputDStream 中的 compute 方法。

由于使用 KafkaInputDStream 的情况比较多,所以分析 ReceiverInputDStream 中 compute 方法 , 调用 ReceiverTracker 中的 getBlocksOfBatch 获取到这段时间内从所有的ReceiverBlockInfo, 最后生成 BlockRDD , 改 RDD 的分区是通过有多少的 BlockId 确定的

```
 val receiverTracker = ssc.scheduler.receiverTracker
        val blockInfos = receiverTracker.getBlocksOfBatch(validTime).getOrElse(id, Seq.empty)
        val blockIds = blockInfos.map { _.blockId.asInstanceOf[BlockId] }.toArray

        // Register the input blocks information into InputInfoTracker
        val inputInfo = InputInfo(id, blockInfos.flatMap(_.numRecords).sum)
        ssc.scheduler.inputInfoTracker.reportInfo(validTime, inputInfo)

```


3.调用 jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToNumRecords)) 提交生成的jobs 到 JobScheduler 上。把 Job 封装成 JobHandler 在 JobExecutor(线程池) 中执行。 放入 JobScheduler 定义的eventloop 后,主要是用来记录job 运行的时间。最后调用 Job 的run 方法执行。在job run 执行之后 就是 RDD 的 执行逻辑了。

```
private class JobHandler(job: Job) extends Runnable with Logging {
    def run() {
      ssc.sc.setLocalProperty(JobScheduler.BATCH_TIME_PROPERTY_KEY, job.time.milliseconds.toString)
      ssc.sc.setLocalProperty(JobScheduler.OUTPUT_OP_ID_PROPERTY_KEY, job.outputOpId.toString)
      try {
        eventLoop.post(JobStarted(job))
        // Disable checks for existing output directories in jobs launched by the streaming
        // scheduler, since we may need to write output to an existing directory during checkpoint
        // recovery; see SPARK-4835 for more details.
        PairRDDFunctions.disableOutputSpecValidation.withValue(true) {
          job.run()
        }
        eventLoop.post(JobCompleted(job))
      } finally {
        ssc.sc.setLocalProperty(JobScheduler.BATCH_TIME_PROPERTY_KEY, null)
        ssc.sc.setLocalProperty(JobScheduler.OUTPUT_OP_ID_PROPERTY_KEY, null)
      }
    }
  }

```

#### Checkpoint







