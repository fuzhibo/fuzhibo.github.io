---
layout: post
title:  "使用 jekyll 构建 github pages 的学习笔记整理"
date:   2019-12-11 16:19:29 +0800
categories: jekyll
---
&emsp;&emsp;这里只是将自己使用 Jekyll 构建 github pages 的过程整理一下。  
&emsp;&emsp;首先，当然是在 github 上创建一个自己的账号，然后新建一个 `<username>.github.io` 的仓库，这样 github 就可以把这个仓库识别为 **github pages**，不过就实际来说建立一个其它名字的仓库也是可以设置 **github pages** 的，区别在于其它名字的仓库允许使用 `master 分支下的 docs 目录`作为 **github pages** 访问页面的存放目录，也可以使用其它分支作为访问页面的存放目录。如果只是用户的 **github pages** 那么就只会允许在 `master`分支下创建相关的页面。这个对我初期的搭建造成了极大的困扰，一开始我一直以为只要是 **github pages** 就可以选择其它的目录，直到发现以 `<username>.github.io` 命名的仓库下的相关设置页面下并没有下拉框可以供分支或者 `master/docs` 选择。  
&emsp;&emsp;建立好仓库后，只需要一个 index.html 就可以看到效果了。现在可以使用 `https://<username>.github.io` (同时要在仓库设置选项卡下勾选 `Enforce HTTPS`) 来进行访问。如果想要使用自己的域名，那么就要去购买一个域名然后在相关的域名提供商页面下配置该域名的 A 记录和 CNAME 。同时要在仓库的根目录下新建一个 CNAME 的文件，里面就只填写自己想要使用的域名。
### DNS 的配置：
```console
@          A             185.199.108.153
@          A             185.199.109.153
@          A             185.199.110.153
@          A             185.199.111.153
www      CNAME           username.github.io.
```
### CNAME 文件的配置
```console
custom.domain
```
&emsp;&emsp;更改好配置后，现在可以通过自己自定义的域名来访问页面了。接下来需要思考一下怎么样来搭建自己的博客，github 推荐是用 Jekyll，这是一个专门的将纯文本转换为静态博客网站的的框架，使用 `Ruby` 开发。那么现在就需要去学习如何安装并使用这套框架。  
1. 安装 Ruby（在 ubuntu 上进行安装）
>&emsp;&emsp;使用这个命令进行安装：`sudo apt install ruby-full`；检测 Ruby ：`ruby --version`。安装好 Ruby 之后其自带一个 `gem` 的工具来管理 Ruby 库和程序。默认是通过 `Ruby Gem`（例如 [rubygems.org](http://rubygems.org/)）源来进行查找、安装、升级和卸载软件包。但是国内访问速度很慢，还是要换源。换源方法可以参考 [gems.ruby-china.com](https://gems.ruby-china.com/) 上的教程来完成源的替换。
2. 安装 Bundler
>&emsp;&emsp;这个是能够跟踪并安装所需的特定版本的 gem，以此来为 Ruby 项目提供一致运行环境的工具（给人感觉类似 node 的 npm 工具）。Jekyll 不可避免会使用这个工具，所以需要安装配置它。使用这个命令来安装：`gem install bundler`。装好后在使用的过程中可能会遇到 `'Cant find gem bundler (>= 0.a) with executable bundle'` 这样的异常，解决办法是运行 `gem update --system` 来升级 gem，这其实是 gem 的特定版本的 bug 升级后就可以解决。