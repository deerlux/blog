title: hexo + github 发布静态博客
date: 2014-10-11 23:48:20
tags: Linux
---

网上的教程很多，我这里只讲一下干货，也是自己的一个备忘。

# 安装node.js

以archlinux下为例，其他的发行版请自行查找相关的软件包。

``` sh
pacman -S nodejs
```

# 系统中安装hexo

``` sh
sudo npm install -g hexo
```

# 初始化blog目录

``` sh
cd your/blog/dir
hexo init
```

# 在目录中安装所依赖的库

``` sh
npm install
```

# 修改 ``_config.yml`` 文件

主要是要修改下面几项

``` yml
title:          # 你的博客名称
author:         # 你的名字
language: zh_CN # 设置中文

url: http://deerlux.github.io       # 设置你要发布的地址
root: /blog                         # 设置你要发布地址的子目录

theme: pacman                       # 设置要使用的主题，个人推荐pacman，有很多国内适用的特性。

deploy:                             # 如果你和我一样用github pages发布，请参照
  type: github
  repo: https://github.com/deerlux/blog.git
  branch: gh-pages
  message: Updated by hexo          # 这是git提交的消息，你可以改成你需要的。

```

不过一直没有搞定如何设置github的repo为ssh格式的。

# 安装并设置pacman主题

首先到github上下载此主题。

``` sh
cd your/blog/dir/themes
git clone git@github.com/A-limon/pacman
```

然后进行配置，如果你有新浪微博、多说评论引擎等，可以直接进行配置，改写
themes/pacman/_config.yml文件即可。



