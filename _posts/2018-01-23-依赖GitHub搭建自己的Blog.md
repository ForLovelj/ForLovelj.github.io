---
layout: post
title: 依赖GitHub搭建自己的Blog
author: clow
date: 2018-01-23 21:20:55
categories:
- Blog
tags: Blog
---
-------

这里先看下成品 [clow的Blog](https://forlovelj.github.io)。

之前一直写笔记，后来想自己建一个站写写博客，调研了一系列方案后，选择了Github Pages+Jekyll来依赖GitHub实现自己的Blog，至于好处嘛，那就是省事，省去了自己建站带来的各种开销和维护问题，坏处就是生成的只是静态页面，不支持在线编辑。

关于Github Pages
-------

Github Pages，我们用它提供的服务来搭建自己的博客，它就相当于一个免费的服务器。

*   **好处：**
    
    完全免费、不需要维护、随意DIY主题，默认Https，可绑定自己的域名，总之简单的操作之后，你只管写你的文章就好。
    
*   **限制：**
    
    仓库（也就是你的网站）不要超过1G，不要频繁提交更新（<=10次/小时），每个月带宽上限100G。总之作为一个小型个人博客来说完全够用，如果不够用，那么恭喜你已经小有成就，花点钱自己搭建的服务也是小case了。
    

关于Jekyll
--------

有了服务器，我们用什么来写自己的blog界面呢？以前用过WordPress或者其他前端框架的都知道，要有一个框架来写，要不然自己纯手工造轮子有多麻烦。

**Jekyll**就是这样一个框架，可以去官网([中文](http://jekyllcn.com/)/[English](https://jekyllrb.com/))自行学习如何搭建环境、创建网站、寻找模板资源等等。

*   **好处：**
    
    Github Pages原生支持的框架，生成的是纯静态网站，上手简单，出问题会少一点，解决问题也会快一点。除了官网的资源外，还可以在Github上找到很多开源的模板。
    
*   **限制**：
    
    其实也没什么限制，就是官网的模板说实话，风格偏国外风，想挑出几个能接受的还真不多，当然我们也可以选择其他静态模板框架，比如[Hexo](https://hexo.io/zh-cn/)、[Hugo](https://www.gohugo.org/)、[Pelican](https://www.pelican.com/us/en/)、[Gridea](https://gridea.dev/)，使用方法可以参考官方的，也可以自己搜一些博客学学，再或者找一些[H5模板](https://html5up.net/)也可以。
    

关于NexT
------

[NexT](https://simpleyyt.com/jekyll-theme-next/)，就是我这个网站选择的一个Jekyll的模板，它之前有hexo的版本，后来改写了Jekyll的版本，这个模板的风格相信大家现在已经看到了，话不多说，想自己搭建的就去[官方教程](http://theme-next.simpleyyt.com/getting-started.html#search-system-algolia)看看。

开始搭建Blog
--------

### 登录github

没有注册过的点击[官网](https://github.com/)注册，并登录。

### 新建Repository

1.  ##### 选择New repository
    
    页面右上角有个"+"号，点击后在弹窗中选择New repository
    
2.  ##### 填写Repository name
    
    **注意：** 填写1处的Repository name，格式为xxx.github.io，xxx就是你的用户名，比如我的就是ForLovelj.github.io，xxx必须和你的用户名匹配，也就是说，一个账户只能在Github上创建一个blog仓库。
    
    然后点击Create repository按钮，仓库就创建成功了。
    

### 进入Setting设置GitHub Pages

1.  不出意外的话，界面会自动跳到你的仓库首页，此时选择Setting界面。
    

2.  然后往下翻，会看到GitHub Pages，点击Choose a theme
    
   
3.  进入主题界面，选择一个自己喜欢的主题，点击Select Theme
    
    
4.  随后跳转到一个界面，然后往下翻，点击Commit changes
    

### 查看Blog

到此你的网站就搭建好了，直接访问你之前填写的地址就好了，比如我的ForLoveLj.github.io


然后在自己的Github->Code界面写自己的文章就好了


默认会访问仓库目录下的 index.md，剩下的就是你怎么创建文件的目录结构，怎么访问就好了。


安装 Ruby
------

1.  去[Ruby官网](https://rubyinstaller.org/downloads/)下载并安装Ruby(注意Ruby+Devkit版本号必须一致，安装路径不能有空格)：
    
    ```
    //使用命令查看版本
    $ ruby -version
    ```
    
    
2.  安装```Bundler```：
    
    ```
    $ gem install bundler
    ```
    
    
3.  下载 NexT 主题：
    
    ```
    $ git clone https://github.com/Simpleyyt/jekyll-theme-next.git
    //可以用命令clone到本地，也可以直接下载项目
    ```
    
    因为我们已经创建好了自己的仓库，所以我们把下载好的NexT资源，全部拷贝到我们的仓库目录中去，拷贝到同级，会覆盖之前的一个\_config.yml文件，选择替换。
    
    然后进入我们的仓库目录
    
    ```
    $ cd ForLovelj.github.io
    ```
    
4.  安装依赖(此处可能出现NexT中Gemfile.Lock配置文件中版本不对的问题，按照命令行提示修改)：
    
    ```
    $ bundle install
    ```
    
5.  运行 Jekyll：
    
    ```
    $ bundle exec jekyll server
    ```
    

此时即可使用浏览器访问 ```http://localhost:4000```，检查站点是否正确运行。

之后把代码提交到远程GitHub仓库，就可以在浏览器输入ForLovelj.github.io看见了

**如果需要使用其他模板的话，每个资源都有他们的使用方法，具体参考模板的教程就可以了，大致流程一样，下载资源、替换资源、运行就好了**

写Blog
-----

现在你只需要找一个趁手的 Markdown 编辑器，我目前时直接用的VSCode，安装插件来编写 MarkDown 文件的。

在电脑上编辑你的文章，然后放到仓库项目中的 \_posts 文件夹里，并同步到 GitHub 上即可。

**注意：**

*   \*\*文件格式：\*\*年-月-日-标题.markdown
    
*   **文章内容顶部**：必须有下面的 YAML 头信息：
    
    ```
    ---
    layout: post
    title: Blogging Like a Hacker
    ---
    ```