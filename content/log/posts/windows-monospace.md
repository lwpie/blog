+++
title = 'Monospace 在 Windows 下被渲染成 NSimSun'
date = 2021-09-06
draft = false
+++

一直在 macOS 上码博客，显示正常体验良好，今天下工在 Windows 笔记本上发现 \<code\> 被渲染为宋体，于是进行一些调查。发现 yinyang theme 没有对 code 自定义 CSS 格式，Safari 对 code 的默认渲染已经达到审美需求，而 Windows 下无论 Chrome 还是 Edge 都缺乏必要的样式表。

通过 Dev Tools 查看 Elements - Styles / Computed，可见 monospace 被渲染为 NSimSun，中文样式接近宋体，英文接近 Courier，而非缺省，因此更新博客的样式表增加以下内容

```css
code { font-family: "Ubuntu Mono" }
```

并引入必要的字体

```html
<link href="//fonts.googleapis.com/css?family=Ubuntu+Mono" rel="stylesheet">
```

另对于代码高亮问题，Harry 一拍脑袋问有没有 highlight.js，一查还真有，于是一并使用

```html
<link rel="stylesheet" href="//cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.2.0/build/styles/default.min.css">
<script src="//cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.2.0/build/highlight.min.js"></script>
<script>hljs.highlightAll();</script>
```
