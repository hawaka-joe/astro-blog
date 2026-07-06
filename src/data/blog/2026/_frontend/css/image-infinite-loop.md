---
title: 图片无限滚动效果的实现
author: Walter
pubDatetime: 2026-07-06T14:00:00Z
featured: false
draft: false
tags:
  - css
  - frontend
description: 使用 CSS background 与 animation 实现图片无限水平滚动效果
---

## 如何实现图片无限滚动效果

代码实现如下

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        .container {
            overflow: hidden;
        }

        .img-ctr {
            background: url("./img.png") repeat-x;
            height: 400px;
            width: 7300px;
            animation: slide 3s linear infinite;
        }

        @keyframes slide {
            0% {
                transform: translate3d(0, 0, 0);
            }

            100% {
                transform: translate3d(-1460px, 0, 0);
            }
        }
    </style>
</head>

<body>
    <div class="container">
        <div class="img-ctr"></div>
    </div>
</body>

</html>
```

## 效果预览

<div class="not-prose image-loop-demo">
  <div class="image-loop-demo__viewport">
    <div class="image-loop-demo__track"></div>
  </div>
</div>

<style>
  .image-loop-demo__viewport {
    overflow: hidden;
    border: 1px solid var(--border, #e5e7eb);
    border-radius: 8px;
  }
  .image-loop-demo__track {
    background: url("/assets/blog/image-loop.png") repeat-x;
    height: 400px;
    width: 7300px;
    animation: image-loop-slide 3s linear infinite;
  }
  @keyframes image-loop-slide {
    0%   { transform: translate3d(0, 0, 0); }
    100% { transform: translate3d(-1460px, 0, 0); }
  }
</style>
