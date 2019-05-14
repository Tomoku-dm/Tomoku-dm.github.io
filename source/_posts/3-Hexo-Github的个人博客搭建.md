---
title: Hexo + Github 的个人博客搭建
date: 2018-08-11 23:35:09
tags:
 - blog
---


## 前言
本篇介绍如何使用 hexo 和 github page 来搭建个人博客， [hexo](https://hexo.io/zh-cn/) 是一个博客框架,支持 markdown，有丰富的插件和主题。github page 是 github 的一个项目，给开发者提供一个免费的没有空间限制的私人页面。


## 配置 github
### 1.创建 github 账号
> https://github.com/ 

### 2.创建一个新项目 new reposity

项目必须要遵守格式：账户名.github.io， 不然接下来会有很多麻烦。并且需要勾选Initialize this repository with a README


### 3. 按照图片输入信息

在建好的项目右侧有个settings按钮，点击它，向下拉到GitHub Pages。开启后访问一下你的链接，应该可以看到默认的GitHub Pages效果

打开链接 ***账户名.github.io*** 可以看到默认的github page 页面

## 配置 Hexo

### 安装 Hexo

* 安装 nodejs 
    安装 nodejs 后才可以安装 hexo
    > http://nodejs.cn/ 

* 安装 git 

    用来将 clone github 上的 hexo 主题
    > https://git-scm.com/downloads 

* 使用 npm 安装 Hexo
    ```
    # npm install -g hexo-cli

    - 一键安装
    ```

### 使用 Hexo

* 初始化 Hexo
    ```
    hexo i blog  //init的缩写 blog是项目名，初始化项目
    cd blog //切换到站点根目录
    hexo g //generetor的缩写   渲染
    hexo s //server的缩写,开启 hexo 服务
    ```
* 打开浏览器输入localhost:4000查看默认页面


   * 看到页面就说明成功了，这个就是hexo默认的博客主题。现在你已经可以在这个主题下写博客了。当然，我是不喜欢这样的，幸好，github上有大量的主题可供选择，这里我选择使用nexT主题。

* 切换主题
   * 在站点根目录输入
   * `git clone https://github.com/theme-next/hexo-theme-next themes/next
`
   * 打开根目录下的配置文件 ***_config.yml*** 修改主题
    ```
    $ vim ../hexo/_config.yml  
    theme: next
    $ vim /themes/next/_congig.yml 
    # Schemes     //next 的三种主题
    #scheme: Muse       # 右边目录，首页居中
    scheme: Mist        # 右边弹目录,首页左上
    #scheme: Pisces     # 桌面目录，文章空间小
    #scheme: Gemini     # 同上，首页博客分割
    # Canvas-nest       # 动态背景
    canvas_nest: false   
    ```
* 清楚缓存，重新渲染
    ```
    $ hexo clean  //清除缓存
    $ hexo g  //重新生成代码
    $ hexo s  //部署到本地

    //然后打开浏览器访问 localhost:4000 查看效果
    ```
    
### 将本地的 hexo 代码上传到 Github Page 上

* 修改hexo站点的配置文件
```
$ vim /hexo/_config.yml
deploy:
    type: git
    repository: https://github.com/Tomoku-dm/Tomoku-dm.github.io
    branch: master
    message: update
```
* 部署
```
$ npm install hexo-deployer-git --save
//先装个插件压压惊
$ hexo d  //  部署的命令

//等一会就好了,这回可以直接访问 Github Page 的网址了
```

### 添加博客文章

文章目录是 ***/blogpath/source/_posts/xxx.md*** 
添加文章后重新渲染及部署
```
$ hexo g //生成静态页面
$ hexo d //发布

```


### 关于 next 主题配置可以参考下面这些链接
https://www.jianshu.com/p/9f0e90cc32c2

hexo 的 _config 可以修改：
- 语言、作者、title、座右铭



