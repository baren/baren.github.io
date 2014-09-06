---
layout: post
title: "octopress syntax"
date: 2014-09-06 10:53:04 +0800
comments: true
categories: 
- octopress
- syntax
---


# 代码高亮语法



> { % codeblock [lang:language] [title] [url] [link text] % }
> code snippet
> { % endcodeblock % }



例子：

{% codeblock Javascript Array Syntax lang:js http://j.mp/pPUUmW MDN Documentation %}
var arr1 = new Array(arrayLength);
var arr2 = new Array(element0, element1, ..., elementN);
{% endcodeblock %}

# 图片语法

```
{ % img [class names] /path/to/image [width] [height] [title text [alt text]] %}
```
例子：
```
{ % img /images/assets/linux.png %}
{ % img left /images/assets/linux.png Place Kitten #2 %}
{ % img right /images/assets/linux.png 150 250 Place Kitten #3 %}
{ % img right /images/assets/linux.png 150 250 'Place Kitten #4' 'An image of a very cute kitten' %}
```
# 多个分类

例子

```
categories:
- CSS3
- Sass
- Media Queries

```

# 显示部分内容

插入

```
<!-- more -->
```
会让文章下面的部分不显示，并会提示一个按钮，来查看未显示的内容。
