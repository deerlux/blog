title: Archlinux下安装nginx、Fastcgi
date: 2015-03-17 20:51:19
tags: 系统运维
---


以前一直用apache作为web服务，但是最近只能在一个上网本上编些程序，配置较差，所以想换nginx作为web服务。Archlinux的WIKI其实是很不错的文档，照着里面的说明很多问题都可以解决，但没想到在配置nginx和fastcgi的时候碰到了一些问题，后来才发现WIKI中有个配置文件错了。本文的配置方法基本与官文WIKI相同，只是完成一个基本的配置，能够使nginx下能够正常调用fastcgi运行相关的PHP程序，但是对于相关的配置文件及一些机理性的东西进行总结。

# nginx-fastcgi的工作机制


