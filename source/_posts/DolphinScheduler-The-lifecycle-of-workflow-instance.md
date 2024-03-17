---
title: DolphinScheduler- 工作流实例的生命周期
date: 2023-05-29 08:51:32
categories: [DolphinScheduler]
tags: [IT, DolphinScheduler]
toc: true
---

> [Apache DolphinScheduler](https://github.com/apache/dolphinscheduler) 是一个分布式易扩展的可视化DAG工作流任务调度开源系统。
> 
> 本文主要讲解关于DS (DolphinScheduler) 工作流实例的生命周期。

## 如何触发一个工作流实例?

Command是统一入口，Master异步处理，将Command转换为工作流实例。
![](/images/workflow_instance_lifecycle/how_to_start_workflow_instance.png)

<!--more-->

## Workflow-DAG解析

DAG构建   ->  初始化数据加载  ->  提交任务节点

![](/images/workflow_instance_lifecycle/build_DAG.png)

## Dispatch-分发

- WorkerGroup：要分发的目标分组
- 轮询策略+心跳机制：如何去最优地分发任务？

![](/images/workflow_instance_lifecycle/heartbeat_and_workergroup.png)

- 逻辑任务组件在Master执行，非逻辑任务组件在Worker执行，更具拓展性。

![](/images/workflow_instance_lifecycle/dispatch_task.png)


## Master和Worker的交互过程

![](/images/workflow_instance_lifecycle/master_and_worker_notify.png)


## 运行态操作

暂停 | 停止 | 超时 | 重试 | 容错

![](/images/workflow_instance_lifecycle/runtime_operation.png)