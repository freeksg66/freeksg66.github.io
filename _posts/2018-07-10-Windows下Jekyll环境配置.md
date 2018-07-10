---
layout:     post
title:      Windows下Jekyll环境配置
subtitle:   在本地轻松调试自己的Blog
date:       2018-07-10
author:     BY
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Jekyll
---

## 介绍
　　Jekyll 是一个简单的博客形态的静态站点生产机器。它有一个模版目录，其中包含原始文本格式的文档，通过 Markdown （或者 Textile） 以及 Liquid 转化成一个完整的可发布的静态网站，你可以发布在任何你喜爱的服务器上。Jekyll 也可以运行在 GitHub Page 上，也就是说，你可以使用 GitHub 的服务来搭建你的项目页面、博客或者网站，而且是完全免费的

　　Jekyll 是一个免费的简单静态网页生成工具，可以配合第三方服务例如： Disqus（评论）、多说(评论) 以及分享 等等扩展功能，Jekyll 可以直接部署在 Github（国外） 或 Coding（国内） 上，可以绑定自己的域名。<a href="https://link.jianshu.com/?t=http://jekyll.bootcss.com/">Jekyll中文文档</a>、<a href="https://link.jianshu.com/?t=https://jekyllrb.com/">Jekyll英文文档</a>、<a href="https://link.jianshu.com/?t=http://jekyllthemes.org/">Jekyll主题列表</a>。

　　Jelly搭建个人博客可以参考这篇文章《<a href="https://www.jianshu.com/p/e68fba58f75c">利用 GitHub Pages 快速搭建个人博客</a>》。

***
本文主要记录Windows下如何配置Jekyll环境，便于本地调试博客

***

### Jekyll环境配置
#### 安装Ruby
下载安装exe,地址：https://rubyinstaller.org/downloads/

![](http://pbmurxnd0.bkt.clouddn.com/downloadRuby.png)

安装的注意点：会自动帮你配置环境变量，安装的位置与你clone到本地的blog项目位置在一个盘。

![](https://upload-images.jianshu.io/upload_images/1195023-e5a69bdde0973466.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/513)

测试是否安装完成

    ruby -v
输出版本信息即安装成功。

#### 安装Devkit
还是一样的地址<a href="https://rubyinstaller.org/downloads/">https://rubyinstaller.org/downloads/</a>

![](http://pbmurxnd0.bkt.clouddn.com/downloadDevKit.png)

1.下载完成后，运行安装包解压到与Ruby安装位置在一个盘的某个文件下下，如：D:\DevKit</li>

2.通过初始化来创建 config.yml 文件。在命令行窗口内，输入下列命令：

    cd “D:\DevKit”

***

    ruby dk.rb init

3.在打开的记事本窗口中，于末尾添加新的一行- D:\Ruby200-x64，保存文件并退出。

4.回到命令行窗口内，审查（非必须）并安装。

    ruby dk.rb review

***

    ruby dk.rb install

#### 安装Jekyll
首先安装gem

    gem -v

***

    gem install jekyll

测试是否安装完毕：

    jekyll -v

进入到你clone的项目目录下，运行jekyll项目（<a href="http://jekyllrb.com/docs/quickstart/">官方文档 http://jekyllrb.com/docs/quickstart/</a>）
    
    jekyll s

其中可能报两个错误

<b>错误1：</b>

![](https://upload-images.jianshu.io/upload_images/1195023-b36b8899925c4601.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/571)

解决方案：

按照提示，安装相关的gem

    gem install jekyll-paginate

<b>错误2：jekyll serve启动报错,error:permission denied -bind(2)</b>

![](https://segmentfault.com/img/bVR9iy?w=549&h=172)

解决方案：

这个是由于127.0.0.1：4000这个端口被占用引起的；

首先找到4000端口被哪个进程占用了

    netstat -aon | findstr "4000"

![](https://segmentfault.com/img/bVSand?w=662&h=572)

发现占用4000端口的进程是2844

最后，关闭冲突的进程（快捷键 ctrl + shift + esc）

在任务管理器的服务tab中将PID为2844的进程停止。

![](https://segmentfault.com/img/bVSan8?w=862&h=512)