---
layout: post
title:  "构建 VS Code Extension 的实践总结"
date:   2019-12-12 14:19:29 +0800
categories: vscode,extension
---
&emsp;&emsp;最近突然想写一个 vs code 插件用来链接 gitlab 方便做一些管理，于是就有了这样一个折腾的笔记。  
&emsp;  
&emsp;&emsp;首先是参考 [Your First Extension](https://code.visualstudio.com/api/get-started/your-first-extension) 这篇文章来一步步的构建自己的插件。但是在启动 debug 的时候遇到了问题，程序会一直卡在 `Building` 的状态。  
&emsp;  
&emsp;&emsp;因为插件本身还是以 npm 作为基础，所以还是得从 package.json 入手调查。最后发现有这么 `scripts` 属性下有这么一个配置：`"watch": "tsc -watch -p ./"`，看起来这里应该就是阻塞点了。本来应该是提供来 npm 自身来进行输入文件的监控，但是 vs code 阻塞在了这里。由于不知道为什么会添加一个 watch 在这里也不明白为什么 vs code debug 的时候会阻塞。现在的解决办法就只有删掉它。删掉之后，一切就可以按照预期执行了。看来原因在于执行 prepublish 的时候出现的问题。  
&emsp;  
&emsp;&emsp;打通了插件的构建流程后，那么就可以正式开始插件的开发了，这里再记录一个 vs code 官方的插件教程[仓库](https://github.com/microsoft/vscode-extension-samples)，可以作为自己开发的参考。
