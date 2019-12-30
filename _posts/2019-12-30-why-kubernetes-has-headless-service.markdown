---
layout: post
title:  "Kubernetes 中的 Headless Service 和 ClusterIp 类型 Service 区别在哪儿"
date:   2019-12-30 15:08:29 +0800
categories: kubernetes,headless service
---
&emsp;&emsp;Kubernetes 中，最容易让人混淆的莫过于 Cluster IP 类型的 Service 和 Headless Service。它们都是用于集群内部服务的访问，但是区别在哪里呢？  
&emsp;  
&emsp;&emsp;首先，直接给出结论：`Cluster IP 类型的 Service 用于无状态的服务`而 `Headless Service 则用于有状态的服务`。  
&emsp;  
&emsp;&emsp;无状态的服务，其中一个代表就是 Deployment，而有状态的服务，代表则是 Stateful Set。  其中，Headless Service 主要是为了 Stateful Set 的网络拓扑状态而服务的。  
&emsp;  
&emsp;&emsp;来看看这两种部署类型下，如果分别对它们使用 `Cluster IP Service` 和 `Headless Service` 会产生什么样的结果：  
&emsp;  
**1.&emsp;Deployment&ensp;+&ensp;Cluster&ensp;IP**
>这种情况下，如果使用 `<svc name>.<namespace>.svc.cluster.local` 的域名格式来进行 DNS 解析的话，那么看到的返回是一个 `VIP`，而这个 `VIP` 的后端往往是通过 `IPTABLES` 或者 `IPVS` 来实现转发负载均衡等的模式。  
&emsp;  
如果使用 `<pod name>.<svc name>.<namespace>.svc.cluster.local` 的域名格式来进行 DNS 解析的话，会报错无法解析。  

```console
[root@k8s-master1 statefulset-nginx]# kubectl exec -it bsybox nslookup redis-master-76fff8f756-89mgw.redis-master.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve 'redis-master-76fff8f756-89mgw.redis-master.default.svc.cluster.local'
command terminated with exit code 1
```

**2.&emsp;Deployment&ensp;+&ensp;Headless**
>这种情况下，如果使用 `<svc name>.<namespace>.svc.cluster.local` 的域名格式来进行 DNS 解析的话，返回的是多个 IP，每一个 IP 都是该 Service 后端映射的一个 Pod。需要客户端自行实现负载功能，比如 `ping` 命令就可以从中随机选择一个 IP 来进行连接。  
&emsp;  
如果使用 `<pod name>.<svc name>.<namespace>.svc.cluster.local` 的域名格式来进行 DNS 解析的话，会报错无法解析。

```console
[root@k8s-master1 ~]# kubectl exec -it busybox nslookup nginx-ss-5cc85fcfcb-8nhw8.nginx-ss.defualt.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve 'nginx-ss-5cc85fcfcb-8nhw8.nginx-ss.defualt.svc.cluster.local'
command terminated with exit code 1
```

**3.&emsp;Stateful&ensp;Set&ensp;+&ensp;Cluster&ensp;IP**
>这种组合一开始就不存在，如果创建会报错 `error: cannot expose a StatefulSet.apps`。

```console
[root@k8s-master1 ~]# kubectl expose statefulset web --port=80 --target-port=8080 --name=web-vip
error: cannot expose a StatefulSet.apps
```

**4.&emsp;Stateful&ensp;Set&ensp;+&ensp;Headless**
>在这种情况下，如果使用 `<svc name>.<namespace>.svc.cluster.local` 的域名格式来进行 DNS 解析的话，返回的是多个 IP，每一个 IP 都是该 Service 后端映射的一个 Pod。需要客户端自行实现负载功能，比如 `ping` 命令就可以从中随机选择一个 IP 来进行连接。  
&emsp;  
如果使用 `<pod name>.<svc name>.<namespace>.svc.cluster.local` 的域名格式来进行 DNS 解析的话，返回的是指定 Pod 对应的容器 IP，这个时候就会使用容器间网络（比如 `flannel`）进行数据传输从而达到访问的目的。

&emsp;  
&emsp; &emsp;从上面的比较可以看出，Headless Service 的作用在于是让 `<pod name>` 加入了域名中，这样才能完成网络拓扑的关系确定。如果这个 Stateful Set 需要被当做一个整体进行访问那么使用 `<svc name>.<namespace>.svc.cluster.local` 的域名格式也可以完成，因为 Stateful Set 对应的集群千变万化，有的可能只是一个读写分离集群；有的可能是一主多从集群。那么它的访问入口也需要根据实际情况进行变化，而 `VIP` 的模式只是将多个 IP 进行平等的负载，所以实在是没有必要用 `VIP` 的模式来实现一个负载均衡，这种情况下将负载交由客户端来决定是一个非常聪明的思路。
