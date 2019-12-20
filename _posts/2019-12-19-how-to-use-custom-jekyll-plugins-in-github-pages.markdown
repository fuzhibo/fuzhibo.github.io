---
layout: post
title:  "如何在 github pages 中使用定制的插件"
date:   2019-12-19 14:18:29 +0800
categories: ruby,jekyll,github pages
---
&emsp;&emsp;在写博客的过程中，需要为自己的文章增加一些流程图的展示。首先想到的就是常用的 `mermaid` 图形库，可惜这个在 jekyll 中并没有直接支持，大部分的解决办法是使用 `jekyll-mermaid` 这个 `jekyll plugins` 来解决问题。不过这个这个插件安装了后使用并没有取得相应的效果。通过对代码的阅读，发现原因在于 `jekyll` 进行模板的分析过程中将这部分代码编程了彻底的文本模式导致的。于是只需要在某段文本的下面直接回车一行这样就可以起一个新行来添加自己的流程图代码块。类似这样：
```console
{% raw %}
{% mermaid %}
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
{% endmermaid %}
{% endraw %}
```
&emsp;&emsp;结果会生成这样的一个流程图：

{% mermaid %}
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
{% endmermaid %}

&emsp;&emsp;接下来的操作就是将这个改动提交到 `github pages` 上，但是问题在于 `github` 出于安全考虑，不允许在 `github pages` 的构建过程中引入自己定制的 `jekyll plugins`，但是这并没有阻止个人上传静态的 html 页面，所以只需要提前构建好然后将构建结果放在 master 分支下就可以规避这个问题。  
&emsp;  
&emsp;&emsp;具体的过程可以参考这个[教程](https://www.cnblogs.com/ihardcoder/p/4479356.html)。不过这个方法的麻烦之处在于每次修改都只能在 dev 分支然后修改完成后还需要进行一系列操作合并到 master 分支解决问题。  
&emsp;  
&emsp;&emsp;不过这个插件过于简单，还需要用到 `Liquid` 的标记语法，在 [Unknown tag error](https://help.github.com/en/github/working-with-github-pages/troubleshooting-jekyll-build-errors-for-github-pages-sites#unknown-tag-error) 中的的如下解释似乎给彻底解决这个问题带来了希望：
> This error means that your code contains an unrecognized Liquid tag.
>
> To troubleshoot, make sure all Liquid tags in the file in the error message match Jekyll's default variables and there are no typos in the tag names. For a list of default varibles, see "Variables" in the Jekyll documentation.
>
> Unsupported plugins are a common source of unrecognized tags. If you use an unsupported plugin in your site by generating your site locally and pushing your static files to GitHub, `make sure the plugin is not introducing tags that are not in Jekyll's default variables`. For a list of supported plugins, see "About GitHub Pages and Jekyll."

&emsp;&emsp;那么不使用 `Tag` 进行标记，通过解析自定义的格式，是不是就可以解决了呢？比如类似 `gitlab` 下的 **markdown** 语法：  
```console
      ```mermaid
      graph TD;
           A-->B;
           A-->C;
           B-->D;
           C-->D;
      ```
```
&emsp;&emsp;要实现这样的小目标，就得从自己开发一个 `jekyll plugins` 开始。先将 `jekyll-mermaid` 插件的基本功能放到自己的新插件中实现，再逐步的完善达到最终想要的结果。  
&emsp;  
&emsp;&emsp;创建一个新的插件 [jekyll-mermaid-diagrams](https://github.com/fuzhibo/jekyll-mermaid-diagrams.git)，其实这个插件实现也简单，最核心的代码就这么一段：
```ruby
module Jekyll
  module Mermaid
    module Diagrams
      class Error < StandardError; end
      # Your code goes here...
      class MermaidChart < Liquid::Block

        def initialize(tag_name, markup, tokens)
          super
        end

        def render(context)
          @config = context.registers[:site].config['mermaid']
          "<script src=\"#{@config['src']}\"></script>"\
          "<script>mermaid.initialize({startOnLoad:true});</script>"\
          "<div class=\"mermaid\">#{super}</div>"
        end
      end
    end
  end
end
Liquid::Template.register_tag('mermaid', Jekyll::Mermaid::Diagrams::MermaidChart)
```
&emsp;&emsp;可以看到，这段代码的作用就是继承了 `Liquid::Block` 这个类并实现了自己的 `render` 方法。不过这里要吐槽一下 bundler 生成的 gem 框架，它默认将包名中的 `'-' ` 分割的字符串视为 `lib` 下的一个目录分级，这样的话会导致引入包的时候会有路径问题；同时，在自己代码实现的文件里也不要引入 `jekyll` 之类的包，而在 `test` 目录下引入，因为这个本来就是一个插件，当置于 `jekyll` 环境后会自动的解决相关的依赖。