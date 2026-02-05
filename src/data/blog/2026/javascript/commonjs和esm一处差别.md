---
title: commonjs和esm一处差别
author: Walter
pubDatetime: 2026-02-05T06:30:00Z
featured: false
draft: false
tags:
  - javascript
description: commonjs 与 esm 在处理导出上的差别
---

## 背景

今天遇到一个问题，我有一个工具模块，大概的代码如下：

```js
let client = null;

init() {
  client = initClient();
}

module.exports = { client, init };
```

在别的文件中使用时：

```js
const someClient = require('@/src/util/some-client')

someClient.init();

someClient.client.someFunc();
```

结果却在调用someFunc时发生了错误，提示 `client` 为 `null`

这让我感觉很奇怪，因为我平时使用 `esm` 比较多，这个因为是老项目所以仍然使用的是 `commonjs`

结果果然与这个有关

## 原因

以下是 `Gemini` 给我的回答

---

<br>

---

**Q: module.exports 是复制了一份保存的变量还是保存的引用?**

A: 这是一个非常经典且关键的问题。直接给结论：**`module.exports` 导出的是引用的拷贝（Copy of the Reference）**。

为了让你彻底理解，我们需要区分**基础类型**和**对象类型**，以及 Node.js 的模块加载机制。

---

### 1. 核心机制：它是如何工作的？

在 Node.js 中，每个文件都被包裹在一个隐藏的函数里。这个函数看起来像这样：

```javascript
(function(exports, require, module, __filename, __dirname) {
  // 你的代码就在这里
});

```

当你给 `module.exports` 赋值时，你是在操作 `module` 这个对象的一个属性。

### 2. 不同类型的表现

#### **如果是基本类型（数字、字符串、布尔值）**

当你导出一个基本类型时，它是**值的拷贝**。一旦导出，模块内部的变化不会影响到外部，反之亦然。

* **模块 A:** `let x = 1; module.exports = x; x = 2;`
* **模块 B:** `require('./A')` 得到的依然是 `1`。

#### **如果是对象类型（Object, Array, Function）**

当你导出一个对象时，导出的是**内存地址的引用**。

* 如果你修改了对象内部的属性，外部通过 `require` 获取的对象也会跟着变。
* 因为大家指向的是堆内存（Heap）中同一个地方。

---

### 3. `exports` vs `module.exports` 的陷阱

这是最容易翻车的地方。`exports` 只是 `module.exports` 的一个**快捷方式（引用）**。

* **正确写法：** `exports.age = 18;` （给原始对象加属性，没问题）
* **错误写法：** `exports = { name: 'Tom' };` （你切断了 `exports` 指向 `module.exports` 的联系，导致导出失败）

### 总结对比

| 类型 | 行为 | 结果 |
| --- | --- | --- |
| **基本类型** | 值传递 | 外部获取后，内外互不影响 |
| **引用类型** | 引用传递 | 外部获取后，修改属性会同步影响 |
| **重新赋值** | 覆盖引用 | 如果你给 `module.exports` 赋了一个新对象，旧引用失效 |

---

**一句话总结：** `require` 得到的是 `module.exports` 当时所指向的那个东西。如果是对象，它就像一把钥匙，大家开的是同一扇门。

---

<br>

---

**Q:esm 可以获取到值改变吗**

A: 这是一个非常棒的进阶问题。简单直接的答案是：**可以。**

与 CommonJS 不同，ESM (ECMAScript Modules) 采用的是**实时绑定（Live Bindings）**。

---

### 1. 核心区别：实时绑定 vs. 值的快照

在 CommonJS 中，当你 `require` 一个变量时，你拿到的是导出那一刻的“快照”或者引用的拷贝。但在 ESM 中，导入和导出变量指向的是**同一个内存槽（Memory Slot）**。

* **CommonJS:** 像是我给了你一张我存折的**照片**（或者是一把钥匙）。
* **ESM:** 像是在墙上开了一个**窗口**，你透过窗口直接看我屋里的那个变量。

### 2. 代码示例演示

让我们看一个具体的例子，你会发现 ESM 的神奇之处：

**counter.js (导出端)**

```javascript
export let count = 1;

export function increment() {
  count++; // 模块内部修改了变量
}

```

**main.js (导入端)**

```javascript
import { count, increment } from './counter.js';

console.log(count); // 输出 1
increment();
console.log(count); // 输出 2！在 CommonJS 中这里依然会是 1

```

> **注意：** 虽然导入端可以“看到”值的改变，但导入端是**只读**的。你不能在 `main.js` 里直接写 `count = 10`，这会导致语法错误（Assignment to constant variable）。

---

### 3. 为什么 ESM 要这么设计？

这种“实时绑定”的设计主要有两个目的：

1. **解决循环依赖（Circular Dependencies）：** 因为是引用绑定，即使模块还没执行完，导出的引用已经存在了，这比 CommonJS 处理循环引用更健壮。
2. **静态分析与 Tree-shaking：** 编译器在编译阶段就能确定模块间的依赖关系，而不是等到运行时。

### 总结对比

| 特性 | CommonJS (`module.exports`) | ESM (`export/import`) |
| --- | --- | --- |
| **机制** | 值的拷贝（或引用拷贝） | **实时绑定 (Live Bindings)** |
| **值更新** | 内部修改后，外部通常感知不到（除非是对象属性） | **内部修改后，外部同步更新** |
| **只读性** | 外部可以随意修改导出的变量 | **导入的变量是只读的 (Read-only view)** |
| **时机** | 运行时加载 | 编译时加载（静态） |

---

**避坑指南：**
虽然 ESM 支持实时绑定，但在实际开发中，过度依赖“隐式修改全局变量”会导致代码难以维护。通常建议通过**函数调用**（如上面的 `increment`）来显式触发状态更新。

你想看看在 Node.js 中如何同时混合使用这两种模块系统，或者想了解它们在性能上的差异吗？

温馨提示：如要解锁所有应用的完整功能，请开启 [Gemini 应用活动记录](https://myactivity.google.com/product/gemini)。

---

## 复现

### commonjs

```js
// util.js
let obj1 = null;
let obj2 = {};
let addFunc = null
let n = 0;

function init() {
    n++; // 无法获取这个改动
    // 导出对象无法获取这个改动
    obj1 = {
        name: 'John',
        age: 20
    }
    obj2.name = 'John';
    obj2.age = 20;
    addFunc = (a, b) => a + b
}

module.exports = {
    obj1,
    obj2,
    addFunc,
    init,
    n
}

// main.js
const util = require('./util');

util.init();

console.log(util.obj1);
console.log(util.obj2);
console.log(util.addFunc);
console.log(util.n);

// 运行结果
// null
// { name: 'John', age: 20 }
// null
// 0
```

### esm

```js
// util.mjs
let obj1 = null;
let obj2 = {};
let addFunc = null
let n = 0;

function init() {
    n++; // 可以获取这个改动
    // esm 可以获取这个改动
    obj1 = {
        name: 'John',
        age: 20
    }
    obj2.name = 'John';
    obj2.age = 20;
    addFunc = (a, b) => a + b
}

export { obj1, obj2, addFunc, init, n }

// main.js
import { obj1, obj2, addFunc, n, init } from './util.mjs';

init();

console.log(obj1);
console.log(obj2);
console.log(addFunc);
console.log(n);

// 运行结果
// { name: 'John', age: 20 }
// { name: 'John', age: 20 }
// [Function: addFunc]
// 1
```
