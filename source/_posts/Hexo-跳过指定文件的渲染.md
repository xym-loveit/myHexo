---
title: Hexo 跳过指定文件的渲染
date: 2018-04-16 14:52:03
categories: hexo笔记
tags: [hexo小技巧]
description: Hexo使用小技巧
---

关于hexo的`_config.yml`配置，官方文档中：

> skip_render：跳过指定文件的渲染，您可使用 glob 表达式来匹配路径。

但并没有说明具体该怎么配置，一番折腾后得以解决：

如果要跳过source文件夹下的`test.html`，可以这样配置：
```
skip_render: test.html

```
注意，千万不要手贱加上个`/`写成`/test.html`，这里只能填相对于source文件夹的相对路径。

如果要忽略source下的test文件夹下所有文件，可以这样配置：
```
skip_render: test/*

```
如果要忽略source下的test文件夹下`.html`文件，可以这样配置：
```

skip_render: test/*.html

```
如果要忽略source下的test文件夹下所有文件和目录，可以这样配置：

```
skip_render: test/**

```
如果要忽略多个路径的文件或目录，可以这样配置：

```
skip_render:
    - test.html
    - test/*
    
```

**Tips:**

[如何不处理source目录下某个子目录的所有文件，仅仅是将其copy到public目录中对应目录？](https://github.com/hexojs/hexo/issues/1146)