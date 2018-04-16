---
title: Hexo在遇到特殊符号时出现解析报错
date: 2018-04-17 00:00:35
categories: hexo笔记
tags: [hexo小技巧]
description: Hexo使用解析问题
---

## hexo 在遇到 "{% raw %}{{{% endraw %}" 符号时出现解析报错

最近在更新一篇文章后，无论是 hexo g 生成，还是 hexo s 预览都会报解析错误， 大致如下，后面还有很长的信息，就不贴了：

```
FATAL Something's wrong. Maybe you can find the solution here: http://hexo.io/docs/troubleshooting.html
Template render error: unexpected token: .
    at Object._prettifyError (D:\workspace\IdeaProjects\myHexo\node_modules\nunjucks\src\lib.js:35:11)
    at Template.render (D:\workspace\IdeaProjects\myHexo\node_modules\nunjucks\src\environment.js:526:21)

```

而把那篇文章移除后一切又是正常的。从错误上来看基本可以判断是模板解析错误，从 unexpected token: . 又看不出来具体是哪里出错，一直找不到原因。

今天查找资料发现有人遇到和我类似的问题，但报的是 `unexpected token: {% raw %}}}{% endraw %}` 的错误。搜索一下我那篇文章，果然有好几处带有 `{% raw %}}}{% endraw %}` 符号。尝试着把几处符号删除，果然正常了。看来问题真的出在 }} 上面。

直接说解决方案吧，参考别人的解决方法是在 }} 中间加一个空格，但因为我的是有部分教程含义的文章，所以并不想这样误导人。于是去 github 上找解决方案。

github 上给出的方法是在需要显示 }} 符号的地方加上 `{% raw %}{% raw %}{% endraw %}{% endraw %}` 标签，标记这部分不需要解析。例如文章中可能会出现 `{{ something }}` 的片段，写成 `{% raw %}{% raw %}{{ something }}{% endraw %}{% endraw %}` 就可以了。

虽然有点麻烦，但也算临时解决了这个问题，这是个已知 bug ，希望后续的版本能修复吧，毕竟使用太多 hexo 专属的标签对博客以后的迁移、改版什么的来说还是很麻烦的。

本文链接: [https://icewing.cc/post/hexo-bug-of-quot.html](https://icewing.cc/post/hexo-bug-of-quot.html)