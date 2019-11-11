# Bulma 学习笔记

## 概念

`Bulma` 的所有修饰符都以 `is-` 或者 `has-` 开头

## 用 `bulma` 创建和控制表单

下面是个标准的 `HTML5` 模版，所有的 `bumla` 都要基于 `HTML5`

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Hello Bulma!</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@0.8.0/css/bulma.min.css">
    <script defer src="https://use.fontawesome.com/releases/v5.3.1/js/all.js"></script>
  </head>
  <body>
  
  </body>
</html>
```

`.hero` 创建一个巨大的横幅，该横幅用于显示特定的内容，该**横幅**也可以选择覆盖页面的整个高度

该组件的基本要求是：

`hero` 作为主要容器

  - `hero-body` 作为直接的子元素，您可以在其中放置所有内容

要使满屏高度[.hero](https://bulma.io/documentation/layout/hero/#fullheight-hero)工作，您还需要一个`hero-head`和一个`hero-foot` class

