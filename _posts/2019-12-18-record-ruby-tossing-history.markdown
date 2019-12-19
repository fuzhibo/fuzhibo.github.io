---
layout: post
title:  "折腾 ruby 的一些日常记录"
date:   2019-12-18 14:18:29 +0800
categories: ruby
---
### 2018-12-18
&emsp;&emsp;突然发现 `gem update --system` 没有作用了，系统环境是 `Ubuntu 18.04.3 LTS`。 一开始是因为遇到了 [Cant find gem bundler (>= 0.a) with executable bundle](https://bundler.io/blog/2019/05/14/solutions-for-cant-find-gem-bundler-with-executable-bundle.html) 的问题。查看了下当前环境中的 gem 版本，是 `2.7.7`。很显然，解决问题的关键始终是升级。不过运行了 `gem update --system` 后，系统却告知已经是最新版本。  
```console
root> gem update --system
Latest version already installed. Done.
```
&emsp;&emsp;既然已经不能使用默认的方式升级，只能指定版本了 `gem update --system '3.0.6'`，经过一系列安装后，升级成功。  

------