---
layout: post
title: spark schedule
tags:  [Spark TaskScheduler]
categories: [Spark]
author: mingtian
excerpt: "spark"
---


* 在 Standalone 模式下


###spark 调度器


PROCESS-LOCAL: 数据在同一个 JVM 中，即同一个 executor 上。这是最佳数据 locality。

NODE-LOCAL 数据在同一个节点上,比如数据在同一个节点的另一个 executor上 或在 HDFS 上，恰好有 block 在同一个节点上。速度比 PROCESS_LOCAL 稍慢，因为数据需要在不同进程之间传递

NO-PREF: 数据从哪里访问都一样快，不需要位置优先

RACK-LOCAL: 数据在同一机架的不同节点上。需要通过网络传输数据，比 NODE_LOCAL 慢

ANY: 数据在非同一机架的网络上，速度最慢



#### 提交任务

```
DAGScheduler.submitMissingTasks
|
TaskSchedulerImpl.submitTasks(TaskSet)
|
创建TaskSetManager 
| 
TaskSetManager.addPendingTask
|
TaskSetManager.computeValidLocalityLevels()

```

创建TaskSetManager 时会传入当前任务的TaskSet，在 TaskSetManager的初始化时会调用addPendingTask 方法，根据DAGScheduler.submitMissingTasks 时 确定的 TaskLocation  把 任务分配到 pendingTasksForExecutor 、pendingTasksForHost、 pendingTasksWithNoPrefs( 没有prefeerLocation 的 任务) 几个 Map 中。
computeValidLocalityLevels 方法是通过判断 最佳位置信息对应的map是否有值，决定当前TaskSet在被调度时 locality level 的 顺序

创建完成TaskSetManager 之后，创建 SchedulableBuilder 默认是 FIFO (FIFOSchedulableBuilder) 的模式。SchedulableBuilder 的内部维护一个 Pool 用来保存需要调度的TaskSetManager 。 

```
 schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)
 |
 rootPool.addSchedulable(manager)
 |
override def addSchedulable(schedulable: Schedulable) {
    require(schedulable != null)
    schedulableQueue.add(schedulable) 
    schedulableNameToSchedulable.put(schedulable.name, schedulable)
    schedulable.parent = this
  }
```

CoarseGrainedSchedulerBackend 中通过DriverEndpoint来同在work 上创建的Executor, DriverEndpint 会周期(spark.scheduler.revive.interval)的给自己发消息 ReviveOffers 执行 makeOffers 方法


```
SparkDeploySchedulerBackend.reviveOffers()
|
CoarseGrainedSchedulerBackend.reviveOffers()
|
DriverEndpoint -> ReviveOffers
|
DriverEndpoint.makeOffers()
|
CoarseGrainedSchedulerBackend.makeOffers()
|
TaskSchedulerImpl.resourceOffers(workOffers)
|
CoarseGrainedSchedulerBackend.launchTasks
```

makeOffers() 计算当前资源，给已经注册的Executor 分配任务，当有新的Executor 注册时也会调用makeOffers() makeOffers 会调用CoarseGrainedSchedulerBackend 的 makeOffers() 获取当前所有可用的Executor 包装成 WorkOffer 传递给 TaskSchedulerImpl.resourceOffers , TaskSchedulerImple#resourceOffers 获取到所有可用的Executor后，汇总可以使用的cpu core。
对Pool 中所有的 TaskSet 根据locality 分配任务。

```
for (taskSet <- sortedTaskSets; maxLocality <- taskSet.myLocalityLevels) {
      do {
        launchedTask = resourceOfferSingleTaskSet( // 以TaskSet 为单位，确定 Exceutor
            taskSet, maxLocality, shuffledOffers, availableCpus, tasks)
      } while (launchedTask)
    }
```
在 每个 TaskSet 内部会调用 resourceOffer(execId, host, maxLocality) 来为给定的 execId 选择合适的任务。


1. 在提交TaskSet 给 TaskSchedulerImpl 调度运行时，如果有可用的WorkOffer
2. 先计算TaskSet 的 LocalityLevel 返回的是一个level 数组
3. 遍历每个WorkOffer 如果 WorkOffer 中所剩余的cores 大于运行 spark.task.cpus 指定的 core 
4. 对于WorkOffer 中的 exeuctorId 通过 TaskSetManager 中的 resourceOffer(executorId,host,maxLocality) 来确定要调度的Task


示例：

 ShuffleMapTask(stageId:0, partitionId:0, prefer:linux4), 
 ShuffleMapTask(stageId:0, partitionId:1, prefer:linux2), 
 ShuffleMapTask(stageId:0, partitionId:2, prefer:linux3), 
 ShuffleMapTask(stageId:0, partitionId:3, prefer:linux3), 
 ShuffleMapTask(stageId:0, partitionId:4, prefer:linux4), 
 ShuffleMapTask(stageId:0, partitionId:5, prefer:linux4)
 
 TaskLocality level 分布
  pendingTasksForExecutor:[]  
  pendingTasksForHost:[linux2 -> ArrayBuffer(1),linux4 -> ArrayBuffer(5, 4, 0),linux3 -> ArrayBuffer(3, 2)]
  pendingTasksWithNoPrefs:[] 
  pendingTasksForRack:[]



第 1 次 调度
 
 --WorkerOffer executorId:0  host:linux4 cores:1--
 
 因为只有linux4 所以给 taskId = 0 分配 NODE-LOCAL 级别
 TaskSetManager 选择任务 task 0.0 in stage 0.0 (TID taskId=0, host=linux4, taskLocality=NODE_LOCAL, execId=0)
 
#### 其中 task 0.0  表示  index = 0  attemp = 0  表示第一个任务，重试次数为 0 , 而 taskId = 0 表示该task运行的是对应 rdd 中 partition = 0 的数据

#### Task 的分配判断的条件是是否有可用的WorkOffer , 有可用的就会执行分配Task的操作, Task 在Executor 上运行完成后，返回状态和数据信息用来判断Task的执行情况，TaskSet 、Stage 是否执行完成 
 
 
第 2 次 调度
 
 --WorkerOffer executorId:2  host:linux3 cores:1--
 
 --WorkerOffer executorId:0  host:linux4 cores:0--
 
 因为 linux4 上没有cores , 只能给 linux3 分配, 在pendingTasks 中 linux3 上可以运行 [2,3] 选择小的运行 NODE-LOCAL 级别
 TaskSetManager 选择任务 task 2.0 in stage 0.0 (TID taskId=1, host=linux3, taskLocality=NODE_LOCAL, execId=2)
 
 
 第 3 次 调度
 
 --WorkerOffer executorId:2  host:linux3 cores:0--
 
 --WorkerOffer executorId:1  host:linux2 cores:1--
 
 --WorkerOffer executorId:0  host:linux4 cores:0--
 
  TaskSetManager 选择任务 task 1.0 in stage 0.0 (TID taskId=2, host=linux2, taskLocality=NODE_LOCAL, execId=1)

第 4 次 调度

 当 linux2 上的 executorId:1 上的 TaskId=2 运行完成，需要重新执行调度
 
 --WorkerOffer executorId:1  host:linux2 cores:1--

 因为 linux2 在 NODE_LOCAL 上没有找到合适的，到下个level 的 pendingTask 都为空，需要在下一个调度级别：ANY 上选择Task
 TaskSetManager 选择任务 task 3.0 in stage 0.0 (TID taskId=3, host=linux2, taskLocality=ANY, execId=1)
 
第 5 次 调度
 
 当 linux3 上 exutorId:2 上的 TaskId=1 运行完成，需要重新执行调度
 
 --WorkerOffer executorId:2  host:linux3 cores:1--

 在 linux3 上可以NODE_LOCAL 级别运行的是 taskId = (2,3) 但是这两个任务已经运行了，只能在 ANY 级别上选择Task
 
 TaskSetManager 选择任务 task 4.0 in stage 0.0 (TID taskId=4, host=linux3, taskLocality=ANY, execId=2)


第 6 次 调度

 当 linux4 上 executorId:0 上的 TaskId=0 运行完成，需要重新执行调度
 
 --WorkerOffer executorId:0  host:linux4 cores:1--

在 linux4 上可以调度的级别是 [NODE_LOCAL,ANY] , 在 NODE_LOCAL 级别上选择任务 TaskId = 5 

TaskSetManager 选择任务 task 5.0 in stage 0.0 (TID taskId=5, host=linux4, taskLocality=NODE_LOCAL, execId=0)


第 7 次 调度

当 linux4 上 executorId:0 上的 TaskId=5 运行完成，需要重新执行调度

--WorkerOffer executorId:0  host:linux4 cores:1--


#### 因为 stgeId:0 中 所有的 ShuffleMapTask 都运行完成了，不会再有新的Task分配。 等待全部ShuffleMapTask 运行完成时，调度StageId:1 中的ResultTask 分配到各个Executor 上执行


stageId:0 运行完成后返回Array[MapStatus] 记录数据的保存位置
Array[MapStatus] 数量: 6 > 
{ CompressedMapStatus: location:BlockManagerId(0, linux4, 47221) compress size:39,41},

{ CompressedMapStatus: location:BlockManagerId(1, linux2, 48003) compress size:39,41},

{ CompressedMapStatus: location:BlockManagerId(2, linux3, 49153) compress size:39,41},

{ CompressedMapStatus: location:BlockManagerId(1, linux2, 48003) compress size:39,41},

{ CompressedMapStatus: location:BlockManagerId(2, linux3, 49153) compress size:39,41},

{ CompressedMapStatus: location:BlockManagerId(0, linux4, 47221) compress size:39,41}


#### stageId:1 中有两个ResultTask 这个是由 spark.default.parallelism 配置决定的，一般为core 的 2~3倍 , 设置为 2 

判断两个ResultTask 的默认执行 level 为 NO_PREF

--WorkerOffer executorId:2  host:linux3 cores:1--

--WorkerOffer executorId:1  host:linux2 cores:1--

--WorkerOffer executorId:0  host:linux4 cores:1--


对于 NO-PREF 的Task 记录的是 PROCESS-LOCAL  不明白为什么不是 NO-PREF

TaskSetManager 选择任务 task 0.0 in stage 1.0 (TID taskId=6, host=linux3, taskLocality=PROCESS_LOCAL, execId=2)

TaskSetManager 选择任务 task 1.0 in stage 1.0 (TID taskId=7, host=linux4, taskLocality=PROCESS_LOCAL, execId=0)








