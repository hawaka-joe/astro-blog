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

## The Basics

1. ts提供类型标注，解决了把一个 string 当做函数调用这列会导致运行出错的异常
2. 对于一些在运行时不会引发异常的错误，ts 同样也会警示，如在一个对象上读取不存在的 key，运行时不会发生错误，但是可能导致潜在的错误，ts 也会报错，以及不同的类型间进行比较等等...
3. ts提供的工具功能，能在 IDE 中带来更丰富的代码提示
4. 当 ts 文件中存在错误导致 tsc 无法编译通过时，js 文件默认会更新，也就是说 tsc 报错的同时吧编译产物也更新了，如果想改变这种行为，可通过`noEmitError`选项来改变

## Everyday Types

1. 重复定义 interface，ts 会组合所有同名定义中的属性

    ```ts
    interface User {
      name: string;
    }

    interface User {
      age: number;
    }
    ```

    得到的 `User` 类型同时包含 `name` 和 `age`

2. 类型断言的两种写法

    ```ts
    const myCanvas = document.getElementById("main_canvas") as HTMLCanvasElement;
    const myCanvas = <HTMLCanvasElement>document.getElementById("main_canvas");
    ```

## Narrowing

1. js 中对`null`执行`typeof`得到的结果是'object'
2. never 类型可以赋值给任何类型；但是，除了 never 本身之外，没有任何类型可以赋值给 never 。这意味着你可以使用类型缩小，并依靠 never 出现来进行 switch 语句中的穷举检查

    当联合类型 `Shape`增加一种类型后，default 分支会报错，这样就能提醒处理新的类型

    ```ts
    type Shape = Circle | Square;
 
    function getArea(shape: Shape) {
      switch (shape.kind) {
        case "circle":
          return Math.PI * shape.radius ** 2;
        case "square":
          return shape.sideLength ** 2;
        default:
          const _exhaustiveCheck: never = shape;
          return _exhaustiveCheck;
      }
    }
    ```

