---
title: Hello World
date: '2022-08-01'
categories:   # 分类
 - 随笔
tags:   # 标签
 - VuePress
 - 图床
 - netlify
publish: true   # 发布
---

<p style="color:#7f7f7f">记录博客搭建及发布和图床配置</p>

<!-- more -->

>你好，世界

## 😙😀😁😉

作为一个优雅的程序员第一篇当然是Hello World咯，仅以此篇记录以下整个博客的搭建过程。

![image-20221027192804828](http://img.sybutterfly.top/img/202211142054259.png)

## 1、关于它

此前笔者搭建过一个wordpress的博客，但其对于md文件的支持并不是很好，无意间看见了尤大的VuePress，这是一个基于vue的文档系统，在md文件的展示上有天然的优势，更是出于对Vue的喜爱，笔者决定转身VuePress

[][关于VuePress][关于VuePress ](https://vuepress.vuejs.org/zh/)

## 2、快速开始

VuePress的使用依赖NodeJs环境，所以需要先安装NodeJs。

官网下载安装即可

```sh
#cmd打开命令行，输入以下命令查看node版本，配置npm镜像源加快下载速度
node -v   #出现版本号即为安装成功
npm config set registry https://registry.npm.taobao.org  #淘宝源
```



本项目的开展采用了vuepress-theme-reco主题，这是一款简洁而优雅的 vuepress 博客 & 文档 主题，更多关于该主题的特性可在官网查看。

[vuepress-theme-reco (recoluan.com)](https://vuepress-theme-reco.recoluan.com/)

这里咱们直接采用官网提供的demo项目即可（这里笔者采用的是1.x的分支demo），地址

https://github.com/vuepress-reco/vuepress-theme-reco-demo/tree/demo/1.x



Git Clone后在本地VS打开

在终端输入以下命令即可启动项目

![image-20221027195627742](http://img.sybutterfly.top/img/202211142054044.png)

> 这里由于笔者已经修改了一些主题，所以显示会不一样。

![image-20221027195645007](http://img.sybutterfly.top/img/202211142053608.png)



至此，这个博客的架子已经搭建好了，这个样子对于笔者来说已经很舒服了，可以说是简约爱好者福音。

那么更多的个性化设置，可以参考VuePress官网和vuepress-reco官网，这里笔者再推荐一个主题，**[vuepress-theme-hope](https://github.com/vuepress-theme-hope/vuepress-theme-hope)**，这是一个具有强大功能的 vuepress 主题✨，更多的是它提供了很多的组件可供复用。



## 3、配置静态网站托管

这里有很多方案

### 3.1、GithubPage

这里提供一篇参考[部署 VuePress 到 GitHub Pages (超详细) - 简书 (jianshu.com)](https://www.jianshu.com/p/1f199ee49e4c)

### 3.2、netlify

这是一个静态网页的发布神器。在它的主页上可以看到它的宣传语：**用最快的方式构建最快的网站**。确实快，而且免费，并且访问速度比GithubPage快很多，相对于**Vercel**（宁一个静态托管平台）并不会被qiang。

虽然有免费就是最贵的说法，但现在笔者还是相信这个互联网世界还是有公平和正义存在的🤣🤣🤣🤣🤣。

官网地址：https://app.netlify.com/



那么怎么把项目托管到该平台呢？

首先你需要在Github上新建一个仓库，将自个儿的代码提交到仓库中（这里不支持gitee），而后用GitHub账号登录Netlify

![image-20221027201714262](http://img.sybutterfly.top/img/202211142054086.png)



> 从GitHub中导入你的项目

![image-20221027202055733](http://img.sybutterfly.top/img/202211142054188.png)

> 选择你的博客项目

![image-20221027202147933](http://img.sybutterfly.top/img/202211142054490.png)

> 确定打包命令和发布目录

![image-20221027202246580](http://img.sybutterfly.top/img/202211142054758.png)

> 打包命令可以是

```sh
yarn build 
#或者
npm run build
```

> 发布目录，这取决于你的配置文件，所以这里你的发布目录应该是public

![image-20221027202556989](http://img.sybutterfly.top/img/202211142054532.png)

>确认好信息之后即可打包，完成后将在项目名下出现netlify给你分配的地址

![image-20221027202958702](http://img.sybutterfly.top/img/202211142054442.png)

>如果你不喜欢这个名儿，可以在Domain setting设置中修改，或者绑定自己的域名

![image-20221027203301777](http://img.sybutterfly.top/img/202211142054409.png)

至此，你就可以通过域名直接访问这个项目了，并且在你推送新的代码到GitHub后，他也会自动更新。

## 4、配置图床

> 需要图床和**PicGo**（上传图片的工具）

大家都可能都遇到过这样的问题，把md文件发给朋友后，对方看不到你文件中的图片，只能看到图片的地址，当然你可以将图片打包一起发出，但这样并不优雅。所有更加推荐图床。

**图床：**就是专门存储图片的服务器，上传后你就可以通过直链访问。

这里你可以使用阿里云、腾讯云提供的OOS服务，但是当然是白嫖咯。这里的方案以前有GitHub图床，但由于不明力量这样的访问会很慢，或许可以上传gitee？但现在gitee公开的仓库需要审查，很不方便。所以更推荐以下两方案。

### 4.1、PicGo下载

地址 https://github.com/Molunerfinn/PicGo/releases/tag/v2.3.0

下载安装即可

### 4.2、SM.MS图床

https://smms.app/

这是一个老牌的图床，结合typora和picgo进行使用，相比于github更方便国内访问；个人账号免费额度是5G

>设置picgo中的smms

![image-20221027210837183](http://img.sybutterfly.top/img/202211142054763.png)

>token在smms仪表盘获取

![image-20221027211000737](http://img.sybutterfly.top/img/202211142054561.png)

### 4.3、BiliBili图床

这个是聪明的网页发现的，并不是真的图床，需要PicGo插件，相对于smms更快，但是个人认为可能会丢失无价的数据

![image-20221027211159025](http://img.sybutterfly.top/img/202211142054956.png)

>配置插件

![image-20221027211226513](http://img.sybutterfly.top/img/202211142054640.png)

>获取sessdata，需要你登录bilibili网页端，后F12打开控制面板，在如下路径获取

![image-20221027211127541](http://img.sybutterfly.top/img/202211142054276.png)



> 而后你可以在PicGo主动测试是否成功，注意由于SESSDATA是动态的，所以你可能需要手动定时更新

![image-20221027211426736](http://img.sybutterfly.top/img/202211142054021.png)

> 在这里你可以查看上传的图片

![image-20221027212332768](http://img.sybutterfly.top/img/202211142054489.png)

### 4.4、Typora偏好设置

![image-20221027210630266](http://img.sybutterfly.top/img/202211142055338.png)

> 测试，点击验证将进行测试

![image-20221027210755490](http://img.sybutterfly.top/img/202211142055015.png)

> 配置成功后，你可以在Typora一键上传

![image-20221027211946080](http://img.sybutterfly.top/img/202211142055577.png)



>至此本文完结咯

![E09F7235D9747A476BD0B66536D30B2D](http://img.sybutterfly.top/img/202211142055478.jpeg)

::: warning
Thanks for reading！🎉 🎉 🎉 🎉 
:::

