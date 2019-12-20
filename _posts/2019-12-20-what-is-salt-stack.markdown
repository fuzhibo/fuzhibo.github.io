---
layout: post
title:  "IT 设施基础工具系列 1 —— Salt Stack 的介绍"
date:   2019-12-20 10:40:29 +0800
categories: salt stack,python,IT infrastructure
---
&emsp;&emsp;IT 基础设施的建设中，管理工具是非常重要的一环。而针对裸金属设施来说，最常用的管理工具之一莫过于 `Salt Stack`。  
#### 什么是 Salt Stack
&emsp;&emsp;`Salt Stack` 是一个配置管理和编排工具。它使用一个中心化的仓库来提供新的服务器和其它 `IT 基础设施`以及自动化`重复的系统管理`工作和`代码部署`任务。  
&emsp;  
&emsp;&emsp;`Salt Stack` 的核心组件，是一个远程执行引擎，创建一个安全的、双向的以及高速的通信网络。当一个 master 运行的时候，一个启动的 minion 会创建加密哈希并且连接到 master。再使用公钥认证后，minion 可从 master 接收命令。Salt 也可以运行在一个无 master 的 minion 模式。
#### 为什么是 Salt Stack
&emsp;&emsp;`Salt Stack` 可被用于 `DevOps` 组织，使用它从中心代码仓库拉取开发者代码和配置信息，然后推送这些内容到远程服务器。Salt 的用户也可以二次开发 `Salt Stack` 同时也能够下载到其它用户已经贡献到公共仓库的 `prebuilt configurations`。    
&emsp;  
&emsp;&emsp;`Salt Stack` 区别于其它的配置管理和自动化工具的一点就是`速度`（官方自己的宣传）。它使用多线程的设计架构，能同时执行成千上百个任务。在分布式集群通信上使用 `ZeroMQ` 消息中间件，有效的解耦的了网络连接，意味着整个系统的运转不要求持久化的连接。    
&emsp;  
&emsp;&emsp;`Salt Stack` 使用 `slave-master` 架构，并且在里面实现了`推`和`拉`的操作。这是一个基于`消息驱动`的架构，同时带有`自愈`的特性。当然，Salt 也可以工作在 `agent-based` 或者 `agentless` 模式。  
&emsp;  
&emsp;&emsp;`Salt Stack` 由 4 个部分组成：  
* reactors —— 监听 events，当 agent 使用一个安全 shell 来执行命令到一个目标系统。
* minion —— 这个就是 agent，能够可选的安装在目标并使用 python 来推送命令。
* grains —— 提供关于目标系统的信息到 minon。
* pillars —— 配置文件。  
&emsp;  
&emsp;&emsp;Salt 使用 Jinja2 模板引擎来插入条件语句和实现其它设置在 Salt state 和 pillar 文件中。
#### 如何使用 Salt Stack
&emsp;&emsp;`Salt Stack` 分为`开源`和`企业`版。企业版包括一个 GUI 并提供一个轻量的目录访问协议（访问控制）。企业版的 API 比开源版本的 API 拥有更多的特性。同时，企业版也增强了合规性检测能力，包括存储事件到数据库来提供一个审计日志。  
&emsp;  
&emsp;&emsp;如果是开源版本，主要是使用命令行来进行相关的操作。