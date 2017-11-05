---
layout: post
title: "pandoc 如何自定义样式"
date: 2016-01-04 22:38:32 +0800
tags:
  - pandoc
---

# Pandoc

pandoc 是一个 markdown 文档转换工具，可以把 markdown 转换为诸多的格式，可以定制格式，编写过滤器等。
pandoc 支持的格式，可以参考官网：http://www.pandoc.org

pandoc 的语法，请看：[官网英文](http://pandoc.org/README.html) /  [中文翻译](http://elvisw.com/pandoc-markdown.html)

pandoc 过滤器编写，请看：https://github.com/jgm/pandoc/wiki/Pandoc-Filters

# 如何自定义样式

## Html 样式
Html 可以指定 css 文件进行样式修改

**方式一：引入 css 文件**

```css  
table{border-collapse:collapse;border:1px solid #CCC;background:#efefef;}
table caption{text-align:left; background-color:#fff; line-height:2em; font-size:14px; font-weight:bold; }
table th{text-align:left; font-weight:bold;height:26px; line-height:26px; font-size:12px; border:1px solid #CCC;}
table td{height:20px; font-size:12px; border:1px solid #CCC;background-color:#fff;}
```

``` sh
$ pandoc -s -c [cssfile] [mdfile] -o [htmlname]
```

**方式二：页面内置样式（写入 header）**

```css
<style type="text/css">
table{border-collapse:collapse;border:1px solid #CCC;background:#efefef;}
table caption{text-align:left; background-color:#fff; line-height:2em; font-size:14px; font-weight:bold; }
table th{text-align:left; font-weight:bold;height:26px; line-height:26px; font-size:12px; border:1px solid #CCC;}
table td{height:20px; font-size:12px; border:1px solid #CCC;background-color:#fff;}
</style>
```

```sh
$ pandoc -s -H [cssfile] [mdfile] -o [htmlname]
```

## Docx 样式
pandoc 可以使用 docx 模板进行渲染（注意：模板是修改样式，而不是内容）
> 可以把自己修改后的样式保存为「word 模板」

``` sh
$ pandoc --reference-docx=[模板路径] [mdfile] -o [docxname]
```
### Docx 模板
分享自己的 docx 模板（不断更新中） [技术文档docx模板](http://pan.baidu.com/s/1pLrNOq3)

### TeX 模板
见 [Phodal Huang 毕业设计模板](https://www.phodal.com/blog/pandoc-template-tex-pandoc/) [(百度盘备份)](http://pan.baidu.com/s/1eQX70mm)

使用方法：需要保存为*.docx 文件，然后使用 `--template` 选项指定模板
