---
layout: post
title: spark schedule
tags:  [Spark TaskScheduler]
categories: [Spark]
author: mingtian
excerpt: "spark"
---



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
 rootPool.addSchedulable(manager)
 |
override def addSchedulable(schedulable: Schedulable) {
    require(schedulable != null)
    schedulableQueue.add(schedulable) 
    schedulableNameToSchedulable.put(schedulable.name, schedulable)
    schedulable.parent = this
  }
```

```
SparkDeploySchedulerBackend.reviveOffers()
|
CoarseGrainedSchedulerBackend.reviveOffers()
|
DriverEndpoint -> ReviveOffers
|
DriverEndpoint.makeOffers()
|
TaskSchedulerImpl.resourceOffers(workOffers)
|
CoarseGrainedSchedulerBackend.launchTasks
```

makeOffers() 计算当前资源，给已经注册的Executor 分配任务，当有新的Executor 注册时也会调用makeOffers()





在 TaskScheduler#submitTasks 创建 TaskSetManager 对应提交的Tas

TaskSetManager

