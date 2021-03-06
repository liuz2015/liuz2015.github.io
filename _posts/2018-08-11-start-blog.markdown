---
layout:     post
title:      "Liuz2015博客开通了"
subtitle:   "如何快速搭建github博客"
date:       2018-08-11
author:     "liuz2015"
header-img: "img/post-bg-2015.jpg"
tags:
    - 生活
---

> “哇哦，Liuz2015的博客终于开通啦。 ”

### 写在博客开通的时候

我的github博客就这么开通了。

[跳过废话，直接看搭建教程](#build)

回想起来是大四开始入了代码的坑，那是2015年的夏天吧，而现在都已经是2018年的夏天了。已经过去了三年了，作为一个程序员，没有自己的博客，没有写些东西分享自己的想法，总觉得没法在道上混呢。开通了这个博客呢，也就有一个可以分享所思所想所得的地方咯。之后呢，争取养成写博客的习惯，分享学习上工作上生活上的点点滴滴吧，写写记记，希望大家多多指教。

<p id = "build"></p>
---

### 如何快速搭建github博客

先简单介绍一下，我的github博客使用[GitHub Pages](https://pages.github.com/) + [Jekyll](http://jekyllrb.com/)的技术方案，为什么使用这个方案呢？

**优点如下：**

- 方便。使用github的服务进行建站，使用git的更新/提交管理博客，因为写博客才是重点，别把心思用在建站上。博主之前也有去折腾服务器，折腾域名，折腾wordpress啥的，到最后博客网站是整出来了，可是写博客的热情却消磨掉了。
- 简单，快速。github默认采用[Jekyll](http://jekyllrb.com/)作为建站工具，这是一款开源的建站工具，网上的教程/讨论也很多，比如本博客仅用了个把小时就搞定了。
- 个性化。为什么不用csdn/博客园等网站作为自己的博客呢，无非是觉得不好看呗（人嘛，都是看脸的），而Jekyll可以很方便的进行定制，你不想折腾，网上当然有别人造好的轮子，比如本博客就用了[这款主题](https://github.com/Huxpro/huxpro.github.io)。
- 写作体验。**Markdown**给人很好的写作体验，让写博客变成愉快的事情也是很重要的嘛。

既然有以上优点，那么废话不多说，咱们就介绍具体步骤吧。不过介绍之前，博主需要强调两个字——快速，对，本文介绍的核心在于快速，因为在windows系统下配置Jekyll的本地环境等等显得过于繁琐，所以过程中都进行删减，只为了能够最快的得到一个可以写东西的博客网站。

**快速搭建步骤如下：**

- 新建github仓库
- 选择Jekyll主题
- 修改Jekyll配置
- 发布博客

#### 新建github仓库

首先，你当然需要有一个github的账号。

然后新建一个仓库（new repository），将仓库命名为：\<username\>.github.io。这时候，你就可以访问 https://\<username\>.github.io/ 这个网址了，这就是你的博客的地址了。

将这个仓库拉取到本地。

#### 选择Jekyll主题

这时候，你的博客上什么都没有，如果按照一般的搭建方式，我们需要在本地使用Jekyll搭建好一个博客网站，然后上传到仓库中，如果这么做，我们需要在本地（Wondows）：下载ruby，安装jekyll，安装bundler，建立博客网站，开启jekyll服务器预览网站等步骤(详情见[该博文](https://www.cnblogs.com/yehui-mmd/p/6286271.html))，博主当时看完后，觉得实在过于繁琐，便没有在本地搭建Jekyll的环境，而是直接使用别人搭建好的博客网站，我们这里推荐博主现在用的[这款](https://github.com/Huxpro/huxpro.github.io)。

将这款主题克隆/下载到本地。
```
git clone git@github.com:Huxpro/huxblog-boilerplate.git
```
将里面的内容复制到你的博客仓库下，并上传之。

这时候，再次打开你的博客地址，应该有东西了吧，不过貌似样式什么的都还有问题，不慌，我们进入第三步。

#### 修改Jekyll配置

Jekyll的配置文件名为：_config.yml。网站的基本配置都在这个文件里面，打开文件，将配置修改为你的相关信息，该文件的配置有疑问可以查看克隆仓库中的README文件。

修改然后上传，再次打开博客，现在应该可以看到一个漂亮的网站咯。

但是，你还是不满意，上面的图片啊，一些文字啊，我想改怎么办。

这时候，我们就得看一下这个网站的基本结构了。注意，下面的讲解基于HuxBlog这个Jekyll主题。

我们要关注的有：
- _includes。网站页面组成部分，head，nav，foot。
- _layouts。网站页面的模板，有page，post等，如博文使用post模板。
- _posts。存放博文。
- img。存放图片。
- index.html，about.html，tags.html。网站的基本页面。

每个html里都会在顶端有该界面的基本配置，你也可以修改html的具体内容。

如果你想个性化界面的内容，以上说明应该有帮助。

如果你是个爱折腾的人，还想探索一下，那么你可以凭借前端相关技术对网站进行更好的定制哦。博主就不献丑了哈哈。

#### 发布博客

现在，万事俱备，只欠东风。

在_post文件夹下开始写你的第一篇博客吧，具体的格式可以直接参考之前的文章哦。

至此，快速搭建github教程就完结咯。

## 感想

俗话说，万事开头难，这是我的第一篇博客，希望今后再接再厉咯~
