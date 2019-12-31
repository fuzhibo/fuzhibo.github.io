---
layout: post
title:  "Kubernetes 中的 initContainers 字段和 lifecycle 下面 postStart 字段比较"
date:   2019-12-31 09:27:29 +0800
categories: kubernetes,initContainers,postStart
---
&emsp;&emsp;在 Kubernetes 中，`initContainers` 字段和 `lifecycle.postStart` 字段都有表示在`容器启动`这个状态下预先执行命令的含义，它们之间有哪些区别呢？  
&emsp;  
&emsp;&emsp;首先，最直观的区别当然是这两个字段所描述`运行时机不同`。一个是指向容器启动之前，一个指向容器启动之时。同时，在具体的使用上也有一些区别。
### initContainers
&emsp;&emsp;这个容器和常规的容器并没有太大的区别，只是多了如下的约束：
* Init containers 总是运行到结束
* 每个 Init container 必须成功的完成，在下一个开始运行的时候

&emsp;&emsp;如果一个 Pod 的 `Init container` 运行失败，Kubernetes 重复的重启 Pod 直到 `Init container` 成功，如果 Pod 有一个值为 `Never` 的 `restartPolicy`，Kubernetes 就不会重启 Pod。  
&emsp;  
&emsp;&emsp;`Init container` 的状态将会放在 `.status.initContainerStatuses` 中返回（这个和 `.status.containerStatuses` 字段类似）。  
&emsp;  
&emsp;&emsp;`Init containers` 并不能够完全当做常规容器使用。在容器资源限制上，所有的 `Init containers` 将只会使用最高的资源限制（而不会类似常规容器那样可以单独设置）。`Init containers` 不支持 readiness probes。如果定义了多个 `Init containers` 那么 Kubernetes 将会顺序执行它们而不会像常规容器那样并发执行。  
&emsp;  
### Container hooks (lifecycle.postStart)
&emsp;&emsp;这个就是容器启动时的一个 `hook`，Kubernetes 不会保证它一定会在 ENTRYPOINT 之前执行。  
&emsp;  
&emsp;&emsp;如果 `hook` 长时间未执行完成，这个 Pod 的状态就不会变为 `Running`，它将长期保持在 `Terminating` 状态并且随后会被关闭当到达 `terminationGracePeriodSecond` 这个设定的超时时间后。当 `hook` 执行失败，它也会关闭这个容器。  
&emsp;  
&emsp;&emsp;`hook` 的失败会产生一个 `FailedPostStartHook` 事件，可以通过 describe Pod 来观察。  
### 总结
&emsp;&emsp;如果需要使用 `Init containers` 和 Pod 内常规容器进行交互，最常用的办法当然是各自挂载同一个共享目录。但是使用 `lifecycle.postStart` 就不会有这么麻烦。  
&emsp;  
&emsp;&emsp;如果需要严格的保证运行顺序，`Init containers` 无疑是一个好选择，虽然 `lifecycle.postStart` 通过延时或者其它办法也可以达到类似效果，但是这始终不是一个正确的做法。
