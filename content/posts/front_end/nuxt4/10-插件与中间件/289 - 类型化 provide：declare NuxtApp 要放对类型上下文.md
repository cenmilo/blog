---
url: /posts/eeb89cb0c68b8d1d17c03f82b68d6d42/
title: 类型化 provide：declare NuxtApp 要放对类型上下文
date: 2027-01-16T10:00:00+08:00
updated: 2027-01-16T10:00:00+08:00
author: cmdragon

summary:
  摘要：插件 provide 出去的 helper，返回值写法下 Nuxt 会自动带类型；但要手动声明、或在别处也能拿到准确类型，靠的是 declare module '#app' 里扩展 NuxtApp、declare module 'vue' 里扩展 ComponentCustomProperties。这一篇的重点是 typescript 那页补上的一条容易忽略的规则：Nuxt 把类型分成 app/server/shared/node 多个上下文，增补必须放在"对应上下文会被扫到的目录"里（要 app 生效就放 app/），放错位置类型不生效。外加 augment 文件必须带 import/export 这条通用要求。

categories:
  - 前端开发

tags:
  - nuxt4
  - provide
  - 类型
  - tsconfig
  - NuxtApp
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

274 讲过 `provide('hello', ...)` → 取出来是 `$hello`。这一篇只解决一件事：**怎么让 `$hello` 有正确的类型、还不会"加了声明却不生效"。** 后者是新手最常见的坑，根源不在写法、在"文件放哪"。

## 先看默认：自动类型

plugins 那页 "Typing Plugins" 第一句给了好消息：

> If you return your helpers from the plugin, they will be typed automatically; you'll find them typed for the return of `useNuxtApp()` and within your templates.

只要你用 274 的 `return { provide: { hello: ... } }` 写法，`useNuxtApp().$hello` 和模板里的 `$hello` 就自动有类型，不用手写。那为什么还要讲增补？两种情况逼你手动声明：

- 你在别处（如另一个插件里 `useNuxtApp()`）想拿到带类型的版本，而自动类型没覆盖到；
- 你想给注入属性显式定签名（官方给的就是"advanced use-cases"）。

## 手写增补：两个接口都要 declare

plugins 那页给的形状是**两个模块、两个接口**：

```ts [index.d.ts]
declare module '#app' {
  interface NuxtApp {
    $hello (msg: string): string
  }
}

declare module 'vue' {
  interface ComponentCustomProperties {
    $hello (msg: string): string
  }
}

export {}
```

为什么是两个：

- `#app` 的 `NuxtApp`：管 `useNuxtApp().$hello` 这一路的类型。
- `vue` 的 `ComponentCustomProperties`：管模板里直接写 `$hello` 的类型。

少 declare 一个，就有一个取用处没类型。末尾那句 `export {}` 是官方在多处强调的通用要求——augment 一个模块的文件必须含 import/export，否则它是"全局脚本"而非"模块"，`declare module` 的语义就变了（变成覆盖而不是扩展）。

## 真正的坑：增补要放对"类型上下文"

类型写法对、却不生效——八成是文件放错目录。typescript 那页 "Augmenting Types with Project References" 给了规则：

> Since the project is divided into multiple type contexts, it's important to augment types within the correct context to ensure they're properly recognized. TypeScript will not recognize augmentations placed outside these directories unless they are explicitly included in the appropriate context.

背景是 Nuxt 会按 dev/build/prepare 生成多份 tsconfig，把项目切成不同上下文（同页列的）：

- **`.nuxt/tsconfig.app.json`** — `app/` 目录里的应用代码
- **`.nuxt/tsconfig.node.json`** — `nuxt.config.ts` 和其它上下文之外的文件
- **`.nuxt/tsconfig.server.json`** — 服务端代码
- **`.nuxt/tsconfig.shared.json`** — app 与 server 共享的代码（类型、环境无关工具）

于是官方直接给了"增补文件该放哪"的对应表：

> For example, if you want to augment types for the `app` context, the augmentation file should be placed in the `app/` directory.

> - For the `server` context, place the augmentation file in the `server/` directory.
> - For types that are shared between the app and server, place the file in the `shared/` directory.

对本篇的 `$hello`：它是**应用侧**的注入属性（`useNuxtApp()`/模板都用它），所以那个 `declare module '#app'` 的 `.d.ts` 应该落在 `app/` 目录下，才会进 `app` 上下文被认到。你要是把它丢在项目根某个不被 app 上下文包含的位置，TypeScript 就不认——表现正是"声明写了、`$hello` 还是 any/报错"。

## 别手改生成的 tsconfig

顺带钉一条相关的硬提醒（typescript 页的 warning）：

> Nuxt relies on this configuration, and Nuxt modules can extend it as well. For this reason, it is not recommended to modify your `tsconfig.json` file directly, as doing so could overwrite important settings. Instead, extend it via `nuxt.config.ts`.

增补类型靠"放对目录的 `.d.ts`"和 `nuxt.config`，而不是去手改 `.nuxt` 里生成的那些 `tsconfig.*.json`。另外，类型只有在跑过 `nuxt dev`/`nuxt build`/`nuxt prepare` 之后才会生成；刚加完声明没生效，先确认生成文件已重建。

## 自查

- 默认走 `return { provide }` 就自动有类型；要手动声明就两个接口都 declare（`#app` 的 `NuxtApp` + `vue` 的 `ComponentCustomProperties`）。
- 增补 `.d.ts` 必须放对上下文目录：应用侧（含 `$hello`）放 `app/`、服务端放 `server/`、两侧共享放 `shared/`。放错=声明写了不生效。
- 每个 augment 文件末尾要有 `export {}`（或任意 import/export）。
- 别手改 `.nuxt` 生成的 tsconfig；要扩配置走 `nuxt.config`。刚加声明没生效，先重跑 `nuxt prepare` 看类型是否生成。

## 下一篇

下一篇 290 讲中间件和插件的分工：路由中间件能改路由（navigateTo/abort）但"没有当前路由概念"、不作用在 /api，插件挂应用级能力还能反过来注册中间件。

## 本文依据

- plugins 目录（`directory-structure/app/plugins`）：Typing Plugins 一节（"If you return your helpers from the plugin, they will be typed automatically; you'll find them typed for the return of `useNuxtApp()` and within your templates."、`declare module '#app' { interface NuxtApp { $hello } }` + `declare module 'vue' { interface ComponentCustomProperties { $hello } }` + `export {}`）。
- TypeScript（`guide/concepts/typescript`）：Auto-generated Types（生成时机 dev/build/`nuxt prepare`、warning "it is not recommended to modify your `tsconfig.json` file directly ... Instead, extend it via `nuxt.config.ts`"）；Project References（app/node/server/shared 四份 tsconfig 及各自覆盖范围）；"Augmenting Types with Project References"（"augment types within the correct context"、"augment types for the app context ... placed in the app/ directory"、server→`server/`、shared→`shared/`）。
- defineNuxtPlugin（`api/utils/define-nuxt-plugin`）：provide 的返回形状（`return { provide: { ... } }`）。
