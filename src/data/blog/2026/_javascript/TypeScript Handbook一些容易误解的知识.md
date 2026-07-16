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

3. 类型谓词（Type Predicates）

    ```ts
    // ❌ 普通函数返回 boolean，TypeScript 不知道类型缩窄
    function isCat(animal: Animal): boolean {
      return 'meow' in animal
    }
    // ✅ 使用类型谓词
    function isCat(animal: Animal): animal is Cat {
      return 'meow' in animal
    }

    function makeSound(animal: Animal) {
      if (isCat(animal)) {
        animal.meow()  // ✅ 正确！TypeScript 知道这里是 Cat
      } else {
        animal.bark()  // ✅ 这里自动推断为 Dog
      }
    }
    ````

## More on Functions

1. 函数可以有属性，`type GreetFunction = (a: string) => void;`无法包含属性信息，因此可以采取下面的写法

    ```ts
    type DescribableFunction = {
      description: string;
      (someArg: number): boolean;
    }

    function doSomething(fn: DescribableFunction) {
      console.log(fn.description + " returned " + fn(6));
    }
    
    function myFunc(someArg: number) {
      return someArg > 3;
    }
    myFunc.description = "default description";
    
    doSomething(myFunc);
    ```

2. 如何表示一个函数的类型是构造函数呢，使用`new`关键字，如下：

    ```ts
    type SomeConstructor = {
      new(someArg: string): SomeObject
    }
    ```

    这两种可以同时存在

    ```ts
    interface CallOrConstruct {
      (n?: number): string;
      new (s: string): Date;
    }
    ```

3. 下面这个函数是无法通过 ts 校验的，想想为什么

    ```ts
    function minimumLength<Type extends { length: number }>(
      obj: Type,
      minimum: number
    ): Type {
      if (obj.length >= minimum) {
        return obj;
      } else {
        // Type '{ length: number; }' is not assignable to type 'Type'
        return { length: minimum };
      }
    }
    ```

4. 不要在回调函数中使用可选参数，让回调函数接受所有的参数，自己决定要使用哪些参数
5. 如何在 TypeScript 函数中显式声明 this 的类型：

    ```ts
    interface User {
      id: number;
      admin: boolean;
    }

    interface DB {
      filterUsers(filter: () => boolean): User[];
      users: User[]
    }

    const db: DB = {
      users: [
        { id: 1, admin: true },
        { id: 2, admin: false },
        { id: 3, admin: true },
      ] as User[],

      filterUsers: function (filter) {
        return this.users.filter(filter);
      },
    };

    // 正确：使用普通函数，TS 知道这里的 this 指向当前被校验的那个 User
    const admins = db.filterUsers(function (this: User) {
      return this.admin; // 这里的 this 就是上面 filter.call(u) 传进来的 u
    });
    ```

6. 一个声明为返回 void 的函数类型，可以被赋值为一个有返回值的函数

    ```ts
    const src = [1, 2, 3];
    const dst = [0];

    // push 会返回新数组的长度（number），但 forEach 并不在乎
    src.forEach((el) => dst.push(el));
    ```

## Modules

1. 在 ES 模块中，一个没有任何顶级导入或导出声明的文件将被视为脚本，其内容可在全局范围内使用
如果你的文件当前没有任何 import 语句或 export 语句，但你想将其视为模块，请添加以下代码行：

    ```ts
    export {}
    ```
2.