---
title: "Serverless 给任务调度带来的变化及蚂蚁集团落地实践"
author: "雪连"
authorlink: "https://github.com/sofastack"
description: "如何实现任务调度和存量应用的 Serverless 化，本文为大家介绍 Serverless Task 是如何实现这一解决方案。"
categories: "MOSN"
tags: ["Serverless", "任务调度","弹性伸缩","ServerlessTask","Service Mesh"]
date: 2021-03-02T15:00:00+08:00
cover: "https://gw.alipayobjects.com/mdn/rms_95b965/afts/img/A*A4bcQoXf5ZkAAAAAAAAAAAAAARQnAQ"
---

> Serverless Task 是蚂蚁集团在分布式调度和批处理中间件发展而来的解决方案。通过 ServiceMesh 的精细化引流能力，再利用研发框架的“服务分组”配置能力，将 Serverless Task 流量全部收敛在指定的“服务分组”集群内。结合定时任务本身具备的周期、可预测等特点，根据任务执行情况弹性伸缩“服务分组”内的机器资源从而提升资源利用率和系统稳定性。

### 分布式调度在蚂蚁的场景和遇到的问题

在单体架构中，为了解决一台机器在固定的周期间隔执行相同的任务，避免人工干预过多，有了基于 Cron 的单机调度；随着企业级应用的发展和微服务化以及云原生架构的逐渐演进，原先的单体架构逐渐演变为服务化或者云原生架构。在此背景下，既要解决原先单机要解决的定时调度问题，还需要解决任务管理、负载均衡以及高可用、容灾等问题，同时兼顾用户体验的简单高效，分布式调度产品就应运而生。

在蚂蚁域内，分布式调度广泛应用于各个 BU 的业务场景中，举例：如在支付宝上购买基金的用户每天需要计算基金收益，那么就需要在分布式调度的基础上结合批处理的能力，充分利用应用集群的处理能力，完成每一个用户基金净值的收益计算，典型处理场景如下：

![蚂蚁三层分发典型场景](https://gw.alipayobjects.com/mdn/rms_95b965/afts/img/A*UvM2Q6t-RkUAAAAAAAAAAAAAARQnAQ)

为了充分利用集群的能力，业务会采用按照业务各个维度拆分的方式对数据进行分片，然后根据分片原则加载数据，最后尽可能的将数据分散在集群机器上完成每一个用户基金净值的计算。通过类似上述集群执行的方式，结合分布式调度及批处理的能力，可以完成业务的计算诉求，但是由于这部分计算逻辑被原有的应用集群承载，随着业务的发展和数据量的不断增加，就会有如下的问题：

**稳定性问题**：在线流量如 RPC/MSG 等与任务调度流量（简称异步任务流量）在 CPU/MEM/线程池 资源共享而引发的相互抢占导致的稳定性问题。

**资源利用率问题**：异步任务流量最常用 Cron 表达式来描述，对于一天 24 小时只需要运行 7 个小时的任务来说，剩余的 17 个小时的资源就是浪费掉的。

**效率问题**：业务同学在接入任务调度以及对批处理执行的控制，数据量统计、异常归类和动态调整参数等复杂性导致的研发效率较低；发布上线后，对处理速度、资源容量评估以及稳定性投入导致的运维效率较低。

### Serverless Task 解决方案

为了解决上文分布式调度在蚂蚁的多年实践中遇到的问题，我们提供了 Serverless Task 的解决方案。

![Serverless Task 解决方案](https://gw.alipayobjects.com/mdn/rms_95b965/afts/img/A*a3p2RKSemkkAAAAAAAAAAAAAARQnAQ)

**通过故障隔离的方式来提高稳定性**。Serverless Task 在不影响大家研发习惯的前提下，业务同学只需要通过在 PaaS 平台中，在同一个集群下申请“服务分组”集群，这些机器用于达到只承担异步任务的流量，而屏蔽其他在线流量的访问的目的。机器资源承载的流量做区分后，再配合 ServiceMesh 的精细化引流能力能够将流量收敛在指定的“服务分组”内，同时结合框架的“服务分组”配置能力，能够指定 Bean、消息、任务或者服务是否注册或者启动。通过上述的方式，最终能够实现，指定的异步任务收敛在指定的“服务分组”机器资源内，以此实现在线流量和异步任务流量的故障隔离，避免相互影响，而提升系统的整体稳定性。

**通过弹性伸缩的方式来提高资源利用率**。基于可控弹性伸缩 HPA 技术，通过分析任务的 Cron 表达式或者基于 CPU/Mem/线程池等各项正常或者异常指标，能够将隔离在指定“服务分组”内运行异步任务的机器动态调整其 Pod 数量，在满足业务处理诉求的前提下，通过弹性伸缩技术最大化的提升资源利用率。

**提供更加产品化的能力**。Serverless Task 支持处理器编排、支持迭代隔离、自定义参数和自定义限流等能力以提升业务的研发效率；同时，异步任务在故障隔离的基础上，利用弹性伸缩技术动态调整业务的 Pod 资源数量，可以让业务研发同学尽可能少的关注资源而只需关注任务的运行情况，以此极大的提升运维效率。

### Serverless Task 关键能力介绍

* 弹性伸缩技术

上文介绍了 Serverless Task 的解决方案思路，为了真正实现 Serverless ，让业务同学不关注容量只关注业务逻辑，一个很重要的技术能力就是弹性伸缩。

![Serverless Task 弹性伸缩技术](https://gw.alipayobjects.com/mdn/rms_95b965/afts/img/A*nmt-RpDbuJAAAAAAAAAAAAAAARQnAQ)

弹性伸缩技术，通过分析任务的 Cron 表达式或者基于 CPU/Mem 等各项指标计算出来的画像（每个时刻期望的 Pod 副本数量），来确定每个时刻的应该 Ready 的 Pod 数量，当在流量低峰或者任务没有在运行的时间就可以将机器缩容到相对较小的副本数。同时为了能够在最短的时间内恢复业务 Pod 的运行，启动速度是一个至关重要的指标，采用基于 ServiceMesh、JVM Elastic Heap 和内存 swap 的容器保活技术实现极速启动，来保证业务容器再恢复到期望的副本数时，有足够快的速度。

* 任务链路隔离与伸缩能力

通过上面描述的 Serverless Task 的解决方案，能够将异步任务的流量收敛在一个业务集群指定的“服务分组”集群内，并能够在弹性伸缩的基础上充分利用机器资源。但是，这样就会导致新的问题，当上游系统被隔离后，其处理速度和稳定性都会一定程度的增加，但是对下游的调用量就会激增，导致下游出现热点或者稳定性问题。基于此，我们提供了任务链路的解决方案，期望将 Serverless Task 触发异步任务的门面系统以及对下游系统的调用，都能够通过“服务分组”的方式隔离开，期望组合成一条异步任务流量作为入口流量的任务链路，具体的实现方案如下所示。

![Serverless Task 任务链路隔离与伸缩](https://gw.alipayobjects.com/mdn/rms_95b965/afts/img/A*pMabRIWzz78AAAAAAAAAAAAAARQnAQ)

每一个 Serverless Task 在周期触发时都会自动携带染色标识（每一个异步任务的唯一标识），通过在任务调度平台选择当前异步任务要引流到的“服务分组”，就可以完成门面系统的 Serverless Task 到指定“服务分组”的引流。每一次 Serverless Task 调用门面系统均携带染色标识，门面系统紧接着对下游系统再发起调用，门面系统结合控制面的业务引流能力，就可以在控制面对门面系统下发引流规则，并配置流量比例，便可以将门面系统对下游系统的调用也收敛在指定的“服务分组”集群内。以此类推，下游系统如果继续有对后续系统的调用，也可以采用类似的方式推送引流规则来完成指定调用流量的“服务分组”集群收敛，以此来完成一条 Serverless Task 作为入口流量的任务链路的流量隔离，并具有单独的业务语义，比如：批扣链路、计价链路。

在完成了任务链路的隔离之后，由于入口的流量是由异步任务驱动或者触发的，流量是能够通过 Cron 或者运行状态做比较准确的预测，那么在任务链路执行完毕后，就可以将整个链路的资源全部伸缩掉，同样当异步任务的流量高峰到来之前时再扩容出足够的机器资源，以此在隔离出任务链路的同时，还能提升整体任务链路的稳定性以及整个任务链路的资源利用率。

### 总结与展望

通过 ServiceMesh 的精细化引流能力，可以将 Serverless Task 流量收敛在指定的“服务分组”集群内，再利用框架（如 SOFA Boot）的“服务分组”配置能力，控制非期望的 Bean 和服务在“服务分组”集群内关闭，最终就能够做到将 Serverless Task 流量完整的收敛在指定的服务器集群内，达到流量收敛的目的后，再结合定时任务本身具备的周期、可预测等特点，就可以根据任务执行情况弹性伸缩“服务分组”内的机器资源从而提升资源利用率。

鉴于 Serverless Task 给业务带来稳定性和资源利用率提升的业务价值，还能够提升业务的研发效率，我们还会继续支持更多调度以及批处理业务场景的 Serverless 化，如：金融交互文件场景、ODPS 离/在线转换场景和 FTP 文件处理场景，欢迎大家关注。


