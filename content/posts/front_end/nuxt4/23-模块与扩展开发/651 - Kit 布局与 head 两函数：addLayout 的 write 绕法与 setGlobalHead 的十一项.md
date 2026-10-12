---
url: /posts/7df9e4f3b8a96100bcfaa74a5de6f4b1/
title: Kit 布局与 head 两函数：addLayout 的 write 绕法与 setGlobalHead 的十一项
date: 2027-05-17T09:00:00+08:00
updated: 2027-05-17T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：布局与全局 head 各只有一个 Kit 函数。**`addLayout` 的签名是 `(layout, name)` 两个参数**——第一个可以是模板对象或路径字符串（**给字符串时会被转成 `src` 为该值的模板对象**），第二个是注册用的名字（`default`、`custom` 这类）。它有一条必须知道的限制：**`@vitejs/plugin-vue` 不支持虚拟的 `.vue` 文件，所以要在工作时给第一个参数传 `write: true`**，把布局真的写到磁盘上。**`setGlobalHead` 收一个对象，可写十一个键**（`charset`、`viewport`、`meta`、`link`、`style`、`script`、`noscript`、`title`、`titleTemplate`、`bodyAttrs`、`htmlAttrs`），合并方向是**深合并、你提供的值优先**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 布局注册
  - 全局head
  - NuxtKit
  - 虚拟文件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块想给用户的项目加一个布局、或者往所有页面的 head 里塞一条 meta，各有一个函数。这两页都只列了一个条目，但都各有一条容易忽略的限制。

## `addLayout`：把模板注册成布局

这一页的开场句很短：**布局用来包裹你的页面，例如用页头页脚这些常见组件把页面包起来；布局可以用 `addLayout` 工具注册**。

函数定性：**把模板注册为布局并加入 layouts**。

```ts
function addLayout (layout: NuxtTemplate | string, name: string): void
```

第一个参数有两种形态，**给字符串时会被转换成模板对象，`src` 设为那个字符串值**；给对象时它有六个字段（**全部标为非必填**）：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `src` | `string` | `false` | 模板路径；**不给 `src` 就必须给 `getContents`** |
| `filename` | `string` | `false` | 模板文件名；**不给时从 `src` 生成，这种情况下 `src` 变成必需** |
| `dst` | `string` | `false` | 目标文件路径；不给时由 `filename` 与 Nuxt 的 `buildDir` 生成 |
| `options` | `Record<string, any>` | `false` | 传给模板的选项 |
| `getContents` | `(data) => string \| Promise<string>` | `false` | 以 `options` 对象被调用，返回字符串或解析到字符串的 promise；**给了 `src` 时这个函数会被忽略** |
| `write` | `boolean` | `false` | **为 `true` 时模板被写入目标文件；否则只存在于虚拟文件系统** |

第二个参数 `name` 是**注册这个布局所用的名字**（示例给的形态是 `default`、`custom` 这种）。

两个示例分别对应"从文件注册"和"生成内容注册"：

```ts
import { addLayout, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    const { resolve } = createResolver(import.meta.url)

    addLayout({
      src: resolve('templates/custom-layout.ts'),
      filename: 'custom-layout.ts',
    }, 'custom')
  },
})
```

```ts
addLayout({
  write: true,
  filename: 'my-layout.vue',
  getContents: () => `<template>
  <div>
    <header>My Header</header>
    <slot />
    <footer>My Footer</footer>
  </div>
</template>`,
}, 'custom')
```

第二个示例把 `write: true` 放在第一位，不是随手加的——**这一节末尾的 `::warning` 解释了原因**：

> Due to the lack of support for virtual `.vue` files by `@vitejs/plugin-vue`, you can work around this limitation by passing `write: true` to the first argument of `addLayout`.

**`@vitejs/plugin-vue` 处理不了虚拟的 `.vue` 文件**，所以内容生成的 `.vue` 布局**必须落盘**。这解释了为什么这个示例里三个字段都要写：`write` 落盘、`filename` 给名字（因为此时没有 `src` 可推导）、`getContents` 给内容。

生成的布局里包裹方式是 `<slot />`，用户侧启用它靠 `definePageMeta`：

```vue [app/pages/about.vue]
<script setup lang="ts">
definePageMeta({
  layout: 'custom',
})
</script>

<template>
  <div>About Page</div>
</template>
```

**布局名与 `addLayout` 第二个参数逐字对应**，`layout: 'custom'` 指向的就是那个 `'custom'`。

一条 `::note` 处理了一个跨代差异：

> In Nuxt 2 `error` layout can also be registered using this utility. In Nuxt 3+ `error` layout [replaced] with `error.vue` page in project root.

也就是说**在新版本里不要试图用 `addLayout` 注册一个叫 `error` 的布局**——错误页在项目根目录以 `error.vue` 页面存在，不是布局。

## `setGlobalHead`：一次配置作用于所有页面

定性：**为你的 Nuxt 应用设置全局 head 配置**。这个工具让模块**以编程方式配置 meta 标签、链接、脚本以及其他会应用到所有页面的 head 元素**。

合并方向写得很明确：

> The provided head configuration will be merged with any existing head configuration using deep merging, with your provided values taking precedence.

**深合并，而且你提供的值优先**。这一句是它与 `moduleDependencies` 里 `defaults`（用户优先）相反的一处，别记混。

```ts
import type { SerializableHead } from '@unhead/vue/types'

interface AppHeadMetaObject extends SerializableHead {
  charset?: string
  viewport?: string
}

function setGlobalHead (head: AppHeadMetaObject): void
```

类型上能看出两件事：**可写的键继承自 `@unhead/vue` 的 `SerializableHead`**，Kit 只额外补了 `charset` 与 `viewport` 两个。

页面把能用的键列成 11 条，**全部可选、都会与已有配置合并**：

| 键 | 含义 |
| --- | --- |
| `charset` | 文档字符编码 |
| `viewport` | viewport meta 标签配置 |
| `meta` | meta 标签对象数组 |
| `link` | link 标签对象数组（样式表、图标等） |
| `style` | 内联 style 标签对象数组 |
| `script` | script 标签对象数组 |
| `noscript` | noscript 标签对象数组 |
| `title` | 默认页面标题 |
| `titleTemplate` | 格式化页面标题的模板 |
| `bodyAttrs` | 加到 `<body>` 上的属性 |
| `htmlAttrs` | 加到 `<html>` 上的属性 |

四个官方示例各自演示一类：

```ts
setGlobalHead({
  meta: [
    { name: 'theme-color', content: '#ffffff' },
    { name: 'author', content: 'Your Name' },
  ],
})
```

```ts
setGlobalHead({
  link: [
    {
      rel: 'stylesheet',
      href: 'https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap',
    },
  ],
})
```

```ts
setGlobalHead({
  script: [
    {
      src: 'https://cdn.example.com/analytics.js',
      async: true,
      defer: true,
    },
  ],
})
```

```ts
setGlobalHead({
  htmlAttrs: {
    lang: 'en',
    dir: 'ltr',
  },
  bodyAttrs: {
    class: 'custom-body-class',
  },
})
```

**示例里出现的形态是数组项各自一个对象**，而 `script` 那条把 `async` 与 `defer` 写成布尔值。**页面没有给"同一条 meta 与用户已有条目冲突时怎么去重"的规则**——它只说深合并，所以数组的合并行为要自己实测一次再下结论。

`::tip` 给的适用场景是一句：**这对需要往应用 head 里注入全局 meta、样式表或脚本的模块特别有用**。

## 两处的取舍

| 需求 | 做法 | 注意 |
| --- | --- | --- |
| 提供一个可复用的布局 | `addLayout` | 内容是生成的 `.vue` 时**必须 `write: true`** |
| 全局加一条 meta、外链样式表或脚本 | `setGlobalHead` | **你的值优先**，可能盖掉用户写的 |
| 只给某几个页面换布局 | 不是这一页 | 页面侧用 `definePageMeta({ layout })` |

关于第二行的"可能盖掉用户写的"，文档只陈述了优先级、**没有说这是推荐行为**：模块要注入的通常是新增项，而不是与用户争夺同一个键。

这两页都没记载的：布局的 `name` 与用户 `app/layouts/` 里同名文件冲突时谁赢、`setGlobalHead` 在 SSR 与客户端 hydration 之间的产出差别、以及有没有"移除某条已注入 head"的对应函数。

下一篇讲插件那两个函数：`addPlugin` 与 `addPluginTemplate`，以及 `order` 那个 -20 到 20 的建议区间。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/layout
- https://nuxt.com/docs/4.x/api/kit/head
