---
layout: post
title:  "折腾 ruby 的一些日常记录"
date:   2019-12-18 14:18:29 +0800
categories: ruby
---
### 2019-12-18
&emsp;&emsp;突然发现 `gem update --system` 没有作用了，系统环境是 `Ubuntu 18.04.3 LTS`。 一开始是因为遇到了 [Cant find gem bundler (>= 0.a) with executable bundle](https://bundler.io/blog/2019/05/14/solutions-for-cant-find-gem-bundler-with-executable-bundle.html) 的问题。查看了下当前环境中的 gem 版本，是 `2.7.7`。很显然，解决问题的关键始终是升级。不过运行了 `gem update --system` 后，系统却告知已经是最新版本。  
```console
root> gem update --system
Latest version already installed. Done.
```
&emsp;&emsp;既然已经不能使用默认的方式升级，只能指定版本了 `gem update --system '3.0.6'`，经过一系列安装后，升级成功。  

------
### 2019-12-19
&emsp;&emsp;因为需要 jekyll 支持更多的功能，于是开始折腾 jekyll plugins 的开发，在进行开发之前第一步自然是要学会打包。这个打包的步骤如下（参考这个[博客](https://ruby-china.org/topics/26292)）：  
#### 使用 bundler 开始工作
&emsp;&emsp;前几年，都还是用 `gem + rails` 这些组合来构建包：  
```console
$ gem install rails  //安装rails
$ gem install rails -v 4.2.0   //安装指定版本的rails
$ gem search rails  //查找所有名称中含有rails的gem
$ gem search ^rails  //查找所有名称中以rails开头的gem
$ gem search ^rails -d  //查找所有名称中以rails开头的gem，并显示描述
$ gem build package.gemspec  //构建一个gem，就是把你自己写的gem源码，打包成一个.gem文件
$ gem push pack-1.0.gem  //发布到源上，默认是rubygems.org
```
&emsp;&emsp;现在，可以使用 bundler 搞定一切：
```console
$ bundler gem mygem
Creating gem 'mygem'...
MIT License enabled in config
Code of conduct enabled in config
      create  mygem/Gemfile
      create  mygem/lib/mygem.rb
      create  mygem/lib/mygem/version.rb
      create  mygem/mygem.gemspec
      create  mygem/Rakefile
      create  mygem/README.md
      create  mygem/bin/console
      create  mygem/bin/setup
      create  mygem/.gitignore
      create  mygem/.travis.yml
      create  mygem/test/test_helper.rb
      create  mygem/test/mygem_test.rb
      create  mygem/LICENSE.txt
      create  mygem/CODE_OF_CONDUCT.md
```
&emsp;&emsp;如果是第一次运行上面的命令，可能会进入一个交互式的界面让你确认一些选项。自己使用了比较保守的选择，创建一个 `lib` 的同时也会创建相应的 `test` 目录。
#### 修改 `gemspec` 文件
&emsp;&emsp;这个文件描述了这个 Gem 的信息。
```ruby
require_relative 'lib/mygem/version'

Gem::Specification.new do |spec|
  spec.name          = "mygem"
  spec.version       = Mygem::VERSION
  spec.authors       = ["TODO: Write your name"]
  spec.email         = ["TODO: Write your email address"]

  spec.summary       = %q{TODO: Write a short summary, because RubyGems requires one.}
  spec.description   = %q{TODO: Write a longer description or delete this line.}
  spec.homepage      = "TODO: Put your gem's website or public repo URL here."
  spec.license       = "MIT"
  spec.required_ruby_version = Gem::Requirement.new(">= 2.3.0")

  spec.metadata["allowed_push_host"] = "TODO: Set to 'http://mygemserver.com'"

  spec.metadata["homepage_uri"] = spec.homepage
  spec.metadata["source_code_uri"] = "TODO: Put your gem's public repo URL here."
  spec.metadata["changelog_uri"] = "TODO: Put your gem's CHANGELOG.md URL here."

  # Specify which files should be added to the gem when it is released.
  # The `git ls-files -z` loads the files in the RubyGem that have been added into git.
  spec.files         = Dir.chdir(File.expand_path('..', __FILE__)) do
    `git ls-files -z`.split("\x0").reject { |f| f.match(%r{^(test|spec|features)/}) }
  end
  spec.bindir        = "exe"
  spec.executables   = spec.files.grep(%r{^exe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]
end
```
&emsp;&emsp;凡是用 `TODO` 描述的地方，都是需要自行填写。
#### 编写自己的功能代码
&emsp;&emsp;自己的代码在 `lib/mygem.rb` 下实现：
```ruby
require "mygem/version"

module Mygem
  class Error < StandardError; end
  # Your code goes here...
  class HelloToSomeone
    def initialize(someone)
      @someone = someone
    end

    def say()
      puts "Hello, #@someone!"
      return true
    end
  end
end
```
#### 测试自己写的功能代码
&emsp;&emsp;测试的代码在 `test/mygem_test.rb` 下实现：
```ruby
require "test_helper"

class MygemTest < Minitest::Test
  def test_that_it_has_a_version_number
    refute_nil ::Mygem::VERSION
  end

  def test_my_hello_obj
    obj = ::Mygem::HelloToSomeone.new("world")
    refute_nil obj
    obj.say()
  end
end
```
&emsp;&emsp;编写好测试代码后在仓库的根目录下使用 `rake test` 来完成测试。
```console
$ rake test
Run options: --seed 13840

# Running:

Hello, world!
..

Finished in 0.000653s, 3060.7857 runs/s, 3060.7857 assertions/s.

2 runs, 2 assertions, 0 failures, 0 errors, 0 skips
```
#### 打包自己的 gem
&emsp;&emsp;执行 `rake build` 来完成打包的操作：
```console
$ rake build
mygem 0.1.0 built to pkg/mygem-0.1.0.gem.
```
#### 在本地安装这个 gem
&emsp;&emsp;如果只是想在本地安装这个 gem 使用 `rake install` 命令：
```console
$ rake install
mygem 0.1.0 built to pkg/mygem-0.1.0.gem.
mygem (0.1.0) installed.
```
#### 发布这个 gem 到远端
&emsp;&emsp;这里主要是指的是将这个 gem 包发布到 `https://rubygems.org` 上面，当然前提是需要在上面注册一个账号，因为发布的时候需要提交自己的账户密码，类似 github（不得不说，这样的方式是十分有利于社区的快速发展）。  
&emsp;  
&emsp;&emsp;使用 `rake release` 命令：
```console
$ rake release
mygem 0.1.0 built to pkg/mygem-0.1.0.gem.
Tagged v0.1.0.
Pushed git commits and tags.
Pushing gem to https://rubygems.org...
Successfully registered gem: mygem (0.1.0)
```

------
