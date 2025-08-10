---
abbrlink: ''
categories: []
category: 博客
published: 2025-02-01
description: 博客大更新！全新的展示，Designed by Histcat for you.
tags:
- 博客
- blog
title: 博客の更新
---
# 博客大更新！

如果你点进来了这篇文章，那么你一定已经看到了博客的变化了（QWQ）

# 动机

更新博客的动机也比较简单，hexo的miracle主题停止更新了，再加上想要练习一下自己的前端知识，就下决心要改博客

在vue，valaxy，astro之间徘徊了好久，vue因为spa对seo的优化不太好，放弃了；valaxy的文档写的不太完善，看不太懂，最终还是选择了astro

# 过程

显然以我现在的前端知识从头开始写是不太可能的，于是选择了hexo的journal主题作为模板，把它迁移到astro

迁移的过程中参考了astro-paper和Frosti两个主题来编写代码，~~不过最后感觉写出了一坨屎山~~

# 简介

源码全部都放在了`/src`目录下

`/public`目录下只有一个js文件，负责网站的动态行为，包括

1. 宽屏模式下的返回顶部
2. 窄屏模式下抽屉功能的实现
3. 窄屏模式下顶部标题的展现

## `/page`

page目录是astro的路由目录

内含有post目录，负责文章的路由；archives负责归档，category负责分类界面，page负责首页的文章展示，index实现了和`/page/1`一样的功能

## `/style`

存放css文件

## `layout`

负责每一个界面的具体实现

其中BaseLayout是总体的layout实现

## `component`

可以复用的组件，未来的comment也要写在这里

## `utils`

一些工具组件

# 成果

最后的成果就是展示在你面前的博客了，不过到现在还是只完成了一部分，还有好多细节要更新。

如果感兴趣的话，可以看一看[github.com/histcat/blog](https://github.com/histcat/blog)源码，
