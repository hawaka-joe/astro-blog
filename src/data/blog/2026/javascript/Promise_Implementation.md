---
title: Promise 简单实现
author: Walter
pubDatetime: 2026-03-17T10:00:00Z
featured: false
draft: false
tags:
  - javascript
description: Promise手写
---

## 如何实现

代码实现如下：  

```js
class MyPromise {
    constructor(executor) {
        this.state = 'pending';
        this.value = undefined;
        this.reason = undefined;

        this.onFulfilledCallbacks = [];
        this.onRejectedCallbacks = [];

        const resolve = (value) => {
            if (this.state !== 'pending') return;

            if (value instanceof MyPromise) {
                return value.then(resolve, reject);
            }

            this.state = 'fulfilled';
            this.value = value;
            this.onFulfilledCallbacks.forEach(fn => fn())
        }

        const reject = (reason) => {
            if (this.state !== 'pending') return;
            this.state = 'rejected';
            this.reason = reason;
            this.onRejectedCallbacks.forEach(fn => fn())
        }

        try {
            executor(resolve, reject);
        } catch (error) {
            reject(error);
        }
    }

    then(onFulfilled, onRejected) {
        return new MyPromise((resolve, reject) => {
            const handleFulfilled = () => {
                try {
                    const result = onFulfilled ? onFulfilled(this.value) : this.value;
                    if (result instanceof MyPromise) {
                        result.then(resolve, reject);
                    } else {
                        resolve(result);
                    }
                } catch (error) {
                    reject(error);
                }
            };

            const handleRejected = () => {
                try {
                    const result = onRejected ? onRejected(this.reason) : this.reason;
                    resolve(result);
                } catch (error) {
                    reject(error);
                }
            };

            if (this.state === 'fulfilled') {
                queueMicrotask(handleFulfilled);
            } else if (this.state === 'rejected') {
                queueMicrotask(handleRejected);
            } else {
                this.onFulfilledCallbacks.push(() => queueMicrotask(handleFulfilled));
                this.onRejectedCallbacks.push(() => queueMicrotask(handleRejected));
            }
        })
    }
}

const p = new MyPromise((resolve, reject) => {
    setTimeout(() => resolve([1, 2, 3]), 200);
}).then(res => {
    console.log(res);
    return res.map(item => item * 2);
}).then((res) => {
    console.log(res);
    return new MyPromise(resolve => {
        setTimeout(() => resolve(res.map(item => item * 3)), 200);
    });
}).then(res => {
    console.log(res);
});
```
