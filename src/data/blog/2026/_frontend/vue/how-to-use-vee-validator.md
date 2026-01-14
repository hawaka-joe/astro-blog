---
title: 怎么样在 vue3 组合式 API 中使用 vee-validator 进行表单校验
author: Walter
pubDatetime: 2026-01-06T06:00:00Z
featured: true
draft: false
tags:
  - vue
  - frontend
description: 介绍了怎么样在 vue3 组合式 API 中使用 vee-validator 进行表单校验
---

## 如何实现

本文记录下如何使用`vee-validator`进行表单验证  

代码模板如下：  

```html
<form v-show="isLogin">
  <!-- Email -->
  <div class="mb-3">
    <label class="inline-block mb-2">Email</label>
    <input type="email"
      class="block w-full py-1.5 px-3 text-gray-800 border border-gray-300 transition duration-500 focus:outline-none focus:border-black rounded"
      placeholder="Enter Email" />
  </div>
  <!-- Password -->
  <div class="mb-3">
    <label class="inline-block mb-2">Password</label>
    <input type="password"
      class="block w-full py-1.5 px-3 text-gray-800 border border-gray-300 transition duration-500 focus:outline-none focus:border-black rounded"
      placeholder="Password" />
  </div>
  <button type="submit"
    class="block w-full bg-purple-600 text-white py-1.5 px-3 rounded transition hover:bg-purple-700">
    Submit
  </button>
</form>
```

添加表单校验的方法如下：

1. 在`script`中定义校验规则，并定义对应的响应式字段和提交方法  

    ```javascript
    import { useForm } from 'vee-validate';
    import { z } from 'zod';
    import { toTypedSchema } from '@vee-validate/zod';

    const validationSchema = toTypedSchema(
      z.object({
        email: z.string().min(1, '邮箱必填').email('格式无效'),
        password: z.string().min(6, '密码至少 6 位').max(16, '密码最多 16 位'),
      })
    );


    const loginForm = useForm({
      validationSchema,
    });

    const [loginEmail, loginEmailAttrs] = loginForm.defineField('email');
    const [loginPassword, loginPasswordAttrs] = loginForm.defineField('password');

    const submitLogin = loginForm.handleSubmit((values) => {
      console.log('通过校验', values)
    })
    ```

2. `html`中使用 `v-model` 和 `v-bind` 分别绑定值和校验对象，并在下方添加显示错误提示的结构  

    改动点：

      1. `v-model="loginEmail"`
      2. `v-bind="loginEmailAttrs"`
      3. `<span class="text-red-500">{{ loginForm.errors.value.email }}</span>`
      4. 表单提交： `<form v-show="isLogin" @submit.prevent="submitLogin">`

    ```html
    <!-- Login Form -->
    <form v-show="isLogin" @submit.prevent="submitLogin">
      <!-- Email -->
      <div class="mb-3">
        <label class="inline-block mb-2">Email</label>
        <input type="email"
          v-model="loginEmail"
          v-bind="loginEmailAttrs"
          class="block w-full py-1.5 px-3 text-gray-800 border border-gray-300 transition duration-500 focus:outline-none focus:border-black rounded"
          placeholder="Enter Email" />
        <span class="text-red-500">{{ loginForm.errors.value.email }}</span>
      </div>
      <!-- Password -->
      <div class="mb-3">
        <label class="inline-block mb-2">Password</label>
        <input type="password"
          v-model="loginPassword"
          v-bind="loginPasswordAttrs"
          class="block w-full py-1.5 px-3 text-gray-800 border border-gray-300 transition duration-500 focus:outline-none focus:border-black rounded"
          placeholder="Password" />
        <span class="text-red-500">{{ loginForm.errors.value.password }}</span>
      </div>
      <button type="submit"
        class="block w-full bg-purple-600 text-white py-1.5 px-3 rounded transition hover:bg-purple-700">
        Submit
      </button>
    </form>
    ```

## 原理解析

这里重点说明下为什么需要使用`v-bind`绑定`defineField`函数返回元祖的第二个值  

首先解释下 `v-bind` 对象绑定的工作原理：  

**`v-bind` 对象绑定的工作机制**  

`v-bind` 绑定对象时，Vue 会展开对象的所有键值对，并根据键名决定绑定方式：  

### 1. 普通属性（HTML 属性）

如果键名是普通的 HTML 属性名（如 `name`、`id`、`class`），会作为属性绑定：  

```javascript
const attrs = {
  name: 'email',
  id: 'email-input',
  class: 'form-input'
}
```

```vue
<input v-bind="attrs" />
```

等价于：  

```vue
<input :name="attrs.name" :id="attrs.id" :class="attrs.class" />
```

### 2. 事件处理器（以 `on` 开头的属性）  

如果键名以 `on` 开头（如 `onBlur`、`onChange`、`onInput`），Vue 会将其转换为事件监听器：  

```javascript
const attrs = {
  name: 'email',
  onBlur: handleBlur,
  onChange: handleChange,
  onInput: handleInput
}
```

```vue
<input v-bind="attrs" />
```

等价于：

```vue
<input 
  :name="attrs.name"
  @blur="attrs.onBlur"
  @change="attrs.onChange"
  @input="attrs.onInput"
/>

```

## vee-validate 的 `attrs` 对象

`defineField` 返回的 `attrs` 对象通常包含：

```javascript
loginEmailAttrs = {
  name: 'email',           // HTML 属性
  id: 'email',            // HTML 属性
  onBlur: function() {...}, // 事件处理器（转换为 @blur）
  onChange: function() {...}, // 事件处理器（转换为 @change）
  onInput: function() {...}, // 事件处理器（转换为 @input）
  // ... 其他属性
}
```

## 实际例子

假设 `loginEmailAttrs` 是：

```javascript
{
  name: 'email',
  id: 'email-field',
  onBlur: () => console.log('blurred'),
  onChange: () => console.log('changed')
}
```

当你写：

```vue
<input v-bind="loginEmailAttrs" />
```

Vue 会将其展开为：

```vue
<input 
  name="email"
  id="email-field"
  @blur="() => console.log('blurred')"
  @change="() => console.log('changed')"
/>
```

下面是一个完整说明此特性的例子

```vue
<template>
  <div>
    <button 
      v-bind="bindObj" 
      :style="{ backgroundColor: bgColor }"
      class="base-button"
    >
      Hover Me
    </button>
  </div>
</template>

<script setup>
import { ref } from 'vue';

const bgColor = ref('#6366f1'); // 初始颜色

const bindObj = {
  onclick: () => {
    console.log('点击了');
  },
  onmouseenter: () => {
    bgColor.value = '#4338ca'; // 移入变色
    console.log('Hover 开始');
  },
  onmouseleave: () => {
    bgColor.value = '#6366f1'; // 移出恢复
    console.log('Hover 结束');
  }
}
</script>

<style scoped>
.base-button {
  color: white;
  padding: 8px 16px;
  transition: background-color 0.3s;
}
</style>
```
