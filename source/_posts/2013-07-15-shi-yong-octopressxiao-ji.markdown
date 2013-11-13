---
layout: post
title: "Octopress初体验"
date: 2013-07-15 11:54
comments: true
categories: "Octopress"
---
![](http://farm4.staticflickr.com/3796/9654815358_f792c648fb_c.jpg)

最近看到了[Octopress][]，刚好是ruby写的，所以没有理由不试一下。

GitHub提供了博客空间的服务。下面就记录一下我在GitHub上部署Octopress的过程。
<!-- more -->
开始就是照着<http://octopress.org/docs/>上的步骤来呗：
{% codeblock %}
git clone git://github.com/imathis/octopress.git octopress
{% endcodeblock %}
就是把octopress的程序拷到本地的文件夹octopress里，当然可以随便命名。

接着把路径切换到这个文件夹中。

然后就和rails的程序比较像了,安装必要的gem
{% codeblock %}
bundle install
{% endcodeblock %}
最后安装主题
```
rake install
```
感觉把博客放到github还是比较方便的：

首先在github新建个仓库，仓库名需要是`username.github.com`形式的

比如我的用户名是`loveltyoic`，那么我新建的就是`loveltyoic.github.com`。

以下就以这个仓库名为例。


```
rake setup github_pages
```
此时会问你github的仓库地址，那么我的就是`git@github.com:loveltyoic/loveltyoic.github.com.git`

按找教程的说法，这个命令会执行6个作用：

1.  设置仓库地址
2.  将clone时的远程分支origin改为octopress
3.  然后将第一步的仓库地址设为默认远程分支origin
4.  从master切换为source分支
5.  根据仓库地址设置博客地址
6.  在_deploy文件夹下设置master分支，用于部署

稍微解释一下，这个命令会生成一个远程分支origin，其地址就是刚刚输入的仓库地址了，所以这个是可以改的，我之前就是用了https的地址，结果每次都要输入用户名密码，非常麻烦，所以就改成ssh形式的地址。因为我已经设置好了ssh密钥，就可直接push了。

接下来
```
rake generate
rake deploy
```
这两个命令将生成博客页面，然后将页面拷贝到_deploy文件夹下，并且执行git的`add,commit以及push`。这样就会在仓库的master分支下产生博客内容了。

最后可以手动将所有源码push到远程仓库的source分支下。

切换到octopress目录
```
git add .
git commit -m 'xxx'
git push origin source
```
这样一个建立在github上的博客就建好了，接下来就该编辑第一篇博客了，Let's go!

首先说明，所有博客源码都会放在source/_posts文件夹内。
```
rake new_post["我的第一篇博客"]
```
这样就会生成一个markdown格式的源文件，可以用markdown语法编辑啦，括号内就是文章标题。生成的源文件会有一个文件头，在编译时会产生相应的作用，比如关闭评论，添加分类等。

同样可以添加页面
```
rake new_page["xinyemian"]
```
这会在source下创建新的文件夹xinyemian，然后在xinyemian下会自动生成`index.markdown`。
比如为博客添加个“关于我”的页面并放到导航栏中。
首先
```
rake new_page["guan-yu-wo"]
```
然后编辑`source/guan-yu-wo/index.markdown`，加入自我介绍。

在source/_includes/custom/下放的是个性化页面的东东，因此编辑其中的navigation.html就可以把这个关于我页面加入到导航里啦。
如下：
```
<ul class="main-navigation">
  <li><a href="{{ root_url }}/">Blog</a></li>
  <li><a href="{{ root_url }}/blog/archives">Archives</a></li>
  <li><a href="{{ root_url }}/guan-yu-wo/">关于我</a></li>
</ul>
```

在写完一篇博客后，就该生成以及发布了。
```
rake generate
rake preview
```
首先生成文章页面

然后是在本地预览，会启动一个web服务，用浏览器打开<http://localhost:4000>可以预览

如果没有问题，就可以部署到github上了。
```
rake deploy
```
部署过后，访问<http://loveltyoic.github.com>，可以看到博客已经更新了。

以上都是最基本的操作了，随着使用慢慢探索吧。

对了，还有个比较重要的内容就是配置文件。

暂时用到的就是根目录下的_config.yml

可以配置博客标题以及附标题，作者等等，根据注释可以大概看懂。

[Octopress]: http://octopress.org/
