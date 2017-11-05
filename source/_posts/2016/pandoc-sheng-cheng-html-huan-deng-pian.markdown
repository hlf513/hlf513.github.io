---
layout: post
title: "pandoc 生成 Html 幻灯片"
date: 2016-01-07 00:15:38 +0800
tags:
  - pandoc
---

# 背景

利用 markdown 直接生成 [web-based slideshow](https://en.wikipedia.org/wiki/Web-based_slideshow)；可以自定义 css ，足够灵活。

# 支持那些幻灯片框架？

pandoc 包含5种 html 幻灯片框架：

* [DZSlides](https://github.com/paulrouget/dzslides)
* [Slidy](http://www.w3.org/Talks/Tools/Slidy2/)
* [S5](http://meyerweb.com/eric/tools/s5/)
* [Slideous](http://goessner.net/articles/slideous/slideous.html)
* [reveal.js](http://lab.hakim.se/reveal-js)

实际上可以使用任何幻灯片框架（比如[Google I/O HTML5 slide template](https://code.google.com/p/io-2012-slides/)），只要让Pandoc在渲染HTML时使用你指定的模板即可.

# 生成默认模板的幻灯片

pandoc 内置 dzslides 框架

```sh
$ pandoc [*.md] -o [*.html] -t dzslides -s
```

# 可选配置

## 渐进显示

* 生成幻灯片时加入 `-i` 选项，用于控制列表的显示效果（逐条渐入）
* 两段文字显示之间的人为停顿: `...`

## 强制分割

默认是2级标题分割。
可以使用`-----------------`强制分割;也可以使用 `--slide-level` 选项覆盖默认的 Slide level

## 代码高亮风格

控制代码高亮风格的选项有：

* --highlight-style pygments
* --highlight-style kate
* --highlight-style monochrome
* --highlight-style espresso
* --highlight-style haddock
* --highlight-style tango
* --highlight-style zenburn


# 自定义样式

这里着重分析下 **reveal.js** ；为什么？因为只有它有提示板。

## 安装reveal.js

``` sh
$ git clone https://github.com/hakimel/reveal.js
```

## 生成幻灯片 Html

``` sh
$ pandoc [*.md]  -o [*.html] -t revealjs -s [-V theme=beige]
```

支持的样式：

* default：（默认）深灰色背景，白色文字
* beige：米色背景，深色文字
* sky：天蓝色背景，白色细文字
* night：黑色背景，白色粗文字
* serif：浅色背景，灰色衬线文字
* simple：白色背景，黑色文字
* solarized：奶油色背景，深青色文字

## 提示板

按 s 触发；

增加小抄：

``` html
<aside class="notes">

  * 这里是提示1
  * 这里是提示2

</aside>
```

# 参考资料

* [Markdown+Pandoc→HTML幻灯片速成](https://linux.cn/article-4080-1.html)
