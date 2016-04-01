---
layout: post
title: Spark Streaming
tags:  [Spark]
categories: [Spark]
author: mingtian
excerpt: "Spark Streaming"
---

### Spark Streaming

```
  val ssc = new StreamingContext(sparkConf, Seconds(2))
  ssc.checkpoint("checkpoint")
  val lines = KafkaUtils.createStream(ssc, zkQuorum, group, topicMap).map(_._2)
  ssc.start()
  ssc.awaitTermination()
```

#### StreamingContext

1. 启动 DStreamGraph 如果有 Checkpoint 则从 Checkpoint 目录启动
2. 创建 JobScheduler
3. 创建 ContextWaiter
4. 创建 StreamingSource

调用 start 方法后 主要是执行 JobScheduler 的 start 方法



类名 | 作用 | 子类
----|------|----
ReceiverInputDStream | foo  | foo
DStreamGraph | bar  | bar
JobScheduler | baz  | baz
JobGenerator |
ContextWaiter |
Checkpoint |

#### ReceiverInputDStream



#### DStreamGraph

```
val newGraph = new DStreamGraph()
newGraph.setBatchDuration(batchDur_) // 程序中设置的Seconds(10)
newGraph
```
DStreamGraph 持有所有的 InputStream 和 OutputStream 



#### JobScheduler
初始化时创建 JobGenerator 、ReceiverTracker、InputInfoTracker



#### JobGenerator


#### ContextWaiter

#### Checkpoint


* StreamingContext.start() 
   
* Scheduler.start() 

      * ReceiverTracker.start()  创建ReceiverTrackerEndpoint 用来用来管理所有的Receiver、接收Receiver发送的数据										ReceiverExecutor.start() 启动所有的Receiver在各个worker上开始接收数据
	      
	  * JobGenerator.start()  如果没有 checkpoint 执行 startFirstTime() 启动 graph.start(startTime - graph.batchDuration)
	  								graph 启动所有的InputStream、OutputStream 

