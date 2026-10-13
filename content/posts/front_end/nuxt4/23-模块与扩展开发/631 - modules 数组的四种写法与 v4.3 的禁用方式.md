---
url: /posts/206033e9d193f583f09d96751bd5523b/
title: 631 - modules 数组的四种写法与 v4.3 的禁用方式
date: 2027-05-10T10:00:00+08:00
updated: 2027-05-10T10:00:00+08:00
author: cmdragon

summary:
  摘要：`modules` 数组里能放四种东西：**包名、本地文件路径、"路径＋选项对象"的元组、内联模块函数**。配置参考那一节把解析规则写死了——先按 node 的 require 路径在 `node_modules` 里找，**用了 `~~` 别名的再从项目 `rootDir` 解析**，所以 `'./modules/example'` 这种相对写法能不能命中取决于你怎么起头。顺序也是硬规定，原话是"模块按顺序执行，所以顺序很重要"：**先 `nuxt.config.ts` 里声明的，再 `modules/` 目录里发现的，后者按字母序**。禁用是 v4.3 才有的能力，做法不是从数组里删，而是把那个模块的配置键直接设成 `false`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块数组
  - 模块解析
  - 分层
  - 配置写法
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一个模块没生效，屏幕上通常什么都没有——不报错，只是它该注入的东西没出现。这时候第一件事是确认那一行字符串写得对不对。`modules` 数组看着简单，其实有四种形状、两条解析路径和一个规定的执行顺序。

## 四种写法，各解决一件事

概念页给的示例是四个条目，注释就是官方自己的说明。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  modules: [
    // 用包名（官方标注的推荐用法）
    '@nuxtjs/example',

    // 加载本地模块
    './modules/example',

    // 带内联选项地加模块
    ['./modules/example', { token: '123' }],

    // 内联定义的模块
    async (inlineOptions, nuxt) => { },
  ],
})
```

把这四个条目摊开：

| 形状 | 例子 | 什么时候用 | 文档里的说法 |
| --- | --- | --- | --- |
| 包名字符串 | `'@nuxtjs/example'` | 模块装在 `node_modules` 里 | 概念页在这条注释里标了 "recommended usage" |
| 本地文件路径字符串 | `'./modules/example'` | 模块是自己写的、放在项目里 | 概念页注释是 "Load a local module" |
| 元组（字符串＋对象） | `['@nuxtjs/google-analytics', { ua: 'X1234567' }]` | 要给模块传配置 | 配置参考写的是"第一项是模块字符串，第二项是选项对象" |
| 内联模块函数 | `async (inlineOptions, nuxt) => { }` | 只想在配置里写几行逻辑，不想建文件 | 概念页示例给出参数名，配置参考写作 "inline module function" |

配置参考那一节的定义句原文是：**每个模块要么是一个字符串（可以指包名，也可以是文件路径），要么是一个元组（第一项字符串、第二项选项对象），要么是一个内联模块函数**。三选一，没有第五种。

## 字符串到底怎么被找到

这一句只在配置参考那节出现：

> Nuxt tries to resolve each item in the modules array using node require path (in `node_modules`) and then will be resolved from project `rootDir` if `~~` alias is used.

拆成两个动作：

1. **先按 node 的 require 路径解析**，也就是走 `node_modules`。包名写法命中这一步。
2. **如果用了 `~~` 别名，则从项目 `rootDir` 解析**。配置参考自己的示例里就有这一条：`'~~/custom-modules/awesome.js'`，注释写的是 "Relative to your project rootDir"。

这里有个容易被忽略的落差：概念页示例里的本地模块写的是 `'./modules/example'`（相对路径），而配置页示例里写的是 `'~~/custom-modules/awesome.js'`（别名）。**两节用的是两种起头方式**，文档没有解释相对路径以什么为基准。要避开这个不确定，用配置参考那种带 `~~` 的写法更稳——它明确说了从 `rootDir` 解析。

## 顺序是规定，不是习惯

配置参考那节有一段 `::callout`，把顺序写死：

> **Note**: Modules are executed sequentially so the order is important. First, the modules defined in `nuxt.config.ts` are loaded. Then, modules found in the `modules/` directory are executed, and they load in alphabetical order.

三句话三个结论：

- 数组内部**按顺序执行**，所以先后是语义的一部分。
- **`nuxt.config.ts` 里声明的模块先加载**。
- **`modules/` 目录里被自动发现的模块后执行，且按字母序**。

也就是说：即使你不把模块写进数组，放在 `modules/` 目录里的模块也会被加载，而且它们永远排在配置文件里那批之后，位置由文件名决定。**如果一个模块依赖另一个先跑完，把它们分在两处（一个靠目录发现、一个靠数组声明）会失去对顺序的控制**。

顺带一提，这个 `modules/` 目录是自动扫描的入口，和数组里那条 `'./modules/example'` 是同一目录的两种用法。

## 禁用：不是从数组里删

概念页有一节标题带 **v4.3** 标记，做法是把模块对应的**配置键**设成 `false`：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  // 禁用 @nuxt/image 模块
  image: false,
})
```

注意这里改的不是 `modules` 数组，而是顶层的 `image` 键——**"把 `@nuxt/image` 从数组里删掉"和"设 `image: false`"是两件事**。数组里删掉只能去掉你自己声明的那份；而这个键的存在是为了解决另一种情况：**模块是从分层（layer）继承来的**，你的配置文件里根本没有它，没法删。

这一节在概念页，配置参考那一节没有；反过来，配置参考的 `## modules` 那一节只有 **Type: `array`**，**没有 Default 行**，也没提禁用。所以禁用这件事目前只在概念页有记载，它末尾指向分层页里"禁用分层中的模块"那一小节。

## 三个能立刻自查的点

- **元组第二项必须是对象**。`['@nuxtjs/google-analytics', { ua: 'X1234567' }]` 这种形状里选项是第二个元素；配置参考把它描述成"options as a second object"。
- **想知道模块有没有被自动发现**，去看项目根下有没有 `modules/` 目录、里面文件名排序后的位置。它一定排在配置文件声明的那批之后。
- **要关一个从分层来的模块**，只有 `键名: false` 这一条路，且需要 v4.3 以上。版本不够时表现是这个键被当成普通配置读掉，模块照常在跑——文档没有对低版本给出任何提示。

文档没写的部分留个记录：内联函数那个形态，示例给了 `(inlineOptions, nuxt)` 两个参数名，但**两节都没解释 `inlineOptions` 是怎么被填进来的**（元组那种带选项的写法显然对应它，可这条对应关系是推断，不是原文）。

下一篇进入 starter：一条命令生成模块项目，以及它给的四步后续动作。

## 下一篇

下一篇 632：模块 starter：四条创建命令与 dev、test、prepack、release 四个脚本

## 本文依据

- https://nuxt.com/docs/4.x/guide/concepts/modules
- https://nuxt.com/docs/4.x/api/nuxt-config
