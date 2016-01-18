layout: hexo
title: Hexo使用笔记（一）
date: 2015-10-15 20:12:22
tags:
---

使用hexo的初衷很简单，别人家的博客系统要么界面low逼要么内容low逼，一旦有运营介入肯定就有广告。个人并不反对广告，但对于强插的中插广告或者大条幅的广告简直这种强奸眼球的做法是不能忍的。而且更多的目的是边学边用，深入浅出的用一个工具，比直接硬啃html+js+css+balalala....好玩的多。当时选择做程序员，图的就是好玩嘛~

## Hexo简介

Hexo官网的一句概括：（Hexo官网网址：https://hexo.io/）
>Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用[Markdown](http://daringfireball.net/projects/markdown/)（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

**Hexo的本质就是工具集成，集Markdown渲染、插件开发与远程部署的一个炫酷工具。**用来做一个简单优雅的静态博客就非常适合，如果想要炫酷到没朋友请用WordPress~

顺便贴下作者的一篇博客感受下这位ACG爱好者的才（dou）气（bi）：[Hexo 颯爽登場！](http://zespia.tw/blog/2012/10/11/hexo-debut/)

## Hexo安装

首先，你得有个.....
不用女朋友啦别哭，就这些：
[Node.js](http://nodejs.org/)
[Git](http://git-scm.com/)

OK,装完后我们就帅气的用以下命令解决：
`$npm install -g hexo-cli`

然后就搞定了，简直不能再帅...

## Hexo 的HelloWorld

由于先不急着了解Hexo的配置文件与目录结构，先把基本过程过一遍先~工具是先学会使用才去研究原理滴~
```
##建站
$ hexo init <folder>
$ cd <folder>
$ npm install

##生成
$ hexo generate
$ npm install hexo-server --save
$ hexo server
```

在浏览器中打开[http://127.0.0.1:4000](http://127.0.0.1:4000),~duang~
PS：重点说下hexo-server这个坑，由于网上很多教程是针对hexo 2.0，所以3.0以上的童鞋注意下。

## Hexo 部署到gitcafe
OK，本地环境妥妥的了~就可以考虑部署了。我选择gitcafe只因为速度问题大家懂得~

1. 注册gitcafe账号
这个自己整....
2. 新建一个与自己用户名相同名字的项目
不要像我一样为了炫酷取个与用户名不同的项目...否则会很麻烦。gitcafe的pages服务默认就是以用户名同名的项目。
3. 生成gitcafe的SSH key
这样你可以在~/.ssh下看到gitcafe，gitcafe.pub了
```
$ mkdir ~/.ssh
$ ssh-keygen -t rsa -C "你的邮箱@xxx.com" -f ~/.ssh/gitcafe
```
4. 连接配置
首先创建一个配置文件先~	然后加上以下语句：
Host gitcafe.com www.gitcafe.com
IdentityFile ~/.ssh/gitcafe
```
$touch ~/.ssh/config
```

5. gitcafe上设置部署key
将~/.ssh/文件夹下的GitCafe.pub中的内容复制到公钥框中即可。
6. 修改_config.yml
在末尾加上以下语句
```
deploy:
  type: git
  repo: https://gitcafe.com/JHuan/JHuan.git,gitcafe-pages
  ```
  在hexo 3.0 type以后就只有git了没有github和gitcafe之分~
7. 部署！
终于等到你我还好没放弃
`hexo d`
如果一切顺利，你可以通过http://你的用户名.gitcafe.io访问你的博客了！


## 那些坑
如果你一切都顺利那祝贺你可以开始愉快的和hexo玩耍了
如果你也不幸跟我一样踩坑了，看看以下有没能够帮助你的：

1. 记得挂代理....
感谢党感谢人民感谢GFW连npm都要帮我们墙掉那是有多爱我们。
`$npm config set proxy=http://192.168.1.1:8080 `
具体端口和代理地址请自行替换哈
在公司办公网的童鞋记得ssh和git bash也要挂上代理哈

2. 请勿在cmd中使用hexo
如果你也跟我一样还在悲催的使用windows做开发，请记得在gitbash中使用hexo。如若不信你可以试下hexo deploy。


