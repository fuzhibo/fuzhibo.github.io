---
layout: post
title:  "Kubelet drop-in file for systemd 解读"
date:   2020-01-02 11:27:29 +0800
categories: kubernetes,kubelet,systemd,drop-in
---
&emsp;&emsp;这个配置文件决定了 `kubelet` 会如何运行。这个配置文件通过安装包安装到 `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` 下面。
这个文件下面记录的所有文件都被 `kubeadm` 管理。
* `/etc/kubernetes/bootstrap-kubelet.conf` 用于 `TLS Bootstrap`，但是这个只会在 `/etc/kubernetes/kubelet.conf` 不存在的时候被使用。
* `/etc/kubernetes/kubelet.conf` 里面主要是一些 kubelet 特有的定义。
* `/var/lib/kubelet/config.yaml` 里面包含 kubelet 的组件配置 (ComponentConfig) 信息。
* `/var/lib/kubelet/kubeadm-flags.env` 里面包含了 kubelet 启动所需的环境变量。
* `/etc/default/kubelet` (for DEBs) 或者 `/etc/sysconfig/kubelet` (for RPMs)。包含了用户配置的 `kubelet` 环境变量。
