---
title: TypeScript Handbook 查漏补缺
author: Walter
pubDatetime: 2026-02-11T07:51:00Z
featured: false
draft: false
tags:
  - typescript
description: 两年前，我通过阅读官方Handbook的方式第一次了解了 TS，现在我有了两年的 TS 开发经验，回头重读，不知会有什么样的收获，开此帖记录下
---

两年前，我通过阅读官方Handbook的方式第一次了解了 TS，现在我有了两年的 TS 开发经验，回头重读，不知会有什么样的收获，开此帖记录下

### The Basics

1. ts提供类型标注，解决了把一个 string 当做函数调用这列会导致运行出错的异常
2. 对于一些在运行时不会引发异常的错误，ts 同样也会警示，如在一个对象上读取不存在的 key，运行时不会发生错误，但是可能导致潜在的错误，ts 也会报错，以及不同的类型间进行比较等等...
3. ts提供的工具功能，能在 IDE 中带来更丰富的代码提示
4. 当 ts 文件中存在错误导致 tsc 无法编译通过时，js 文件默认会更新，也就是说 tsc 报错的同时吧编译产物也更新了，如果想改变这种行为，可通过`noEmitError`选项来改变