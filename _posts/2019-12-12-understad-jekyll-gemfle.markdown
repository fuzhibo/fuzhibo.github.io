---
layout: post
title:  "理解 Jekyll 项目下的 Gemfile"
date:   2019-12-12 10:07:29 +0800
categories: jekyll,frontend
---
&emsp;&emsp;Jekyll 框架是通过 bundler 安装的，那么这个安装过程的控制就是通过 Gemfile 来完成的，在这个过程中 bundler 做了什么呢？  
&emsp;  
&emsp;&emsp;首先，指定了源 `source`：
```yaml
source "https://rubygems.org"
```
&emsp;  
&emsp;&emsp;接下来，文件设置了 `Gems`：
```yaml
# 这里指定了包 jekyll 并且指定了版本最好是 4.0.0 或者之后的安全版本
# Happy Jekylling!
gem "jekyll", "~> 4.0.0"
# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem "minima", "~> 2.5"
```
&emsp;&emsp;版本运算符一览
>* = Equal To "=1.0"
>* != Not Equal To "!=1.0"
>* &gt; Greater Than ">1.0"
>* < Less Than "<1.0"
>* &gt;= Greater Than or Equal To ">=1.0"
>* <= Less Than or Equal To "<=1.0"
>* ~> Pessimistically Greater Than or Equal To "~>1.0"

&emsp;&emsp;理解 `~>` 操作符，它等价于一个范围的取值操作。
>* gem "my_gem", "~> 1.0" –> gem "my_gem", ">= 1.0", "< 2.0"
>* gem "my_gem", "~> 1.5.0" –> gem "my_gem", ">= 1.5.0", "< 1.6.0"
>* gem "my_gem", "~> 1.5.5" –> gem "my_gem", ">= 1.5.5", "< 1.6.0"

&emsp;&emsp;一个 gem 可以属于一个或者多个 group，当它不属于任何 group 的时候，它被放入了 `:default group`。这里创建了一个专门的 group `jekyll_plugins`。
```yaml
# If you want to use GitHub Pages, remove the "gem "jekyll"" above and
# uncomment the line below. To upgrade, run `bundle update github-pages`.
# gem "github-pages", group: :jekyll_plugins
# If you have any plugins, put them here!
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
end
```
&emsp;&emsp;如果是 `Windows` 和 `JRuby`的平台安装 Jekyll 则还需要安装 tzinfo-data。因为自己是 Ubuntu 作为开发平台，所以就不深入展开了。
```yaml
# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
install_if -> { RUBY_PLATFORM =~ %r!mingw|mswin|java! } do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end
```
&emsp;&emsp;安装 `wdm` 依赖的时候需要注意平台，这个可以通过 `Gem.win_platform` 来进行判断。
```yaml
# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :install_if => Gem.win_platform?
```
&emsp;&emsp;更多详细有关 `Gemfile` 的内容，可以参考这些[文章](https://ruby-china.org/topics/26655)。至于 `Gemfile.lock` 这个文件是使用 `bundle install` 的时候自动生成的文件，具体可以参考这个[文章](https://bundler.io/v1.7/rationale.html)。
