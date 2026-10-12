---
url: /posts/6d63f79d9b78f6cb6e20022d025f2513/
title: Bridge 的定位与第一步：前向兼容层，且两个取数组合式函数用不了
date: 2027-05-05T11:00:00+08:00
updated: 2027-05-05T11:00:00+08:00
author: cmdragon

summary:
  摘要：Bridge 的官方定义是一句：**"Bridge is a forward-compatibility layer that allows you to experience many of the new Nuxt 3 features by simply installing and enabling a Nuxt module."** 它给的是**渐进**："you can make sure your project is (almost) ready for Nuxt 3 and you can gradually proceed with the transition." 但同一页开头的 `::warning` 立刻划了边界：**Bridge 提供与 Nuxt 3 一致的特性、却有限制， notably `useAsyncData` 与 `useFetch` 这两个组合式函数不可用**。第一步也不是装 Bridge，而是**先把 Nuxt 2 升到位**：停掉 `nuxt dev`、**删掉锁文件（`package-lock.json` 与 `yarn.lock`）**、把 nuxt 装到 `^2.17.3`、重装依赖、确认 dev 与 build 都正常；随后才装 `@nuxt/bridge` 与 `nuxi`（dev 依赖）、把配置里的 CommonJS 语法清掉、**并把 `nuxt` 命令全改成 `nuxt2`**。

categories:
  - 前端开发

tags:
  - nuxt4
  - bridge
  - 前向兼容
  - Nuxt2升级
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

手上是一个还在跑的 Nuxt 2 项目，直接上 3 风险太大；老板又想要新特性。Bridge 就是为这种夹缝准备的——但它有两处限制会直接影响日常写法，装之前必须知道。

## 官方定位与两条前置提醒

定性句：**"Bridge is a forward-compatibility layer that allows you to experience many of the new Nuxt 3 features by simply installing and enabling a Nuxt module."** 用途句：**"Using Nuxt Bridge, you can make sure your project is (almost) ready for Nuxt 3 and you can gradually proceed with the transition to Nuxt 3."** ——注意那个 `(almost)`，官方自己就没说"完全等价"。

开头两条提示是这一页最该先读的：

- `::note`：**如果你是要新开一个 Nuxt 3 项目，跳过这一节**，直接去看安装页。
- `::warning`：**"Nuxt Bridge provides identical features to Nuxt 3 … but there are some limitations, notably that `useAsyncData` and `useFetch` composables are not available."**

第二条是硬限制：**新架构里最核心的两个数据获取组合式函数，在 Bridge 下没有**。所以"用了 Bridge 就等于会用到 3 的取数写法"这个预期是不成立的；这一页让你去读后半部分了解细节，而这页之后各节也确实各自补了替代说法（Composition API 那页会讲到 `useFetch` 在旧包里的对应物 `$fetch`/`$fetchState` 换成 `fetch`/`fetchState`）。

## 第一步：先把 Nuxt 2 升到位

`## First Step` 下第一个 `###` 是 Upgrade Nuxt 2，四个动作：

1. **确保 dev server 没在跑（`nuxt dev`）**。
2. **删除所有包锁文件**：原文点名 `package-lock.json` 与 `yarn.lock`。
3. 装最新 Nuxt 2：

```diff [package.json]
- "nuxt": "^2.16.3"
+ "nuxt": "^2.17.3"
```

4. 重装依赖（npm／yarn／pnpm／bun／deno 五种写法都给全）。

之后一条 `::note` 是检查点：**"Once the installation is complete, make sure both development and production builds are working as expected before proceeding."** ——**dev 与 build 都要能过，再往下走。**

## 第二步：装 Bridge 本体

> Install `@nuxt/bridge` and `nuxi` as development dependencies:

```bash
npm install -D @nuxt/bridge nuxi
```

另外四种写法同样齐给（yarn 是 `yarn add --dev`、pnpm/bun 用 `-D`、deno 是 `deno add -D npm:@nuxt/bridge npm:nuxi`）。**两个包都是 dev 依赖**，这一点原文写明。

## 第三步：清掉配置里的 CommonJS

> Please make sure to avoid any CommonJS syntax such as `module.exports`, `require` or `require.resolve` in your config file. **It will soon be deprecated and unsupported.**

替代写法是静态 `import`、动态 `import()` 与 `export default`；**并推荐顺手改名成 `nuxt.config.ts`**（链接指向配置文件那一页）。改完的形状是：

```ts [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  bridge: false,
})
```

**注意 `defineNuxtConfig` 这时是从 `@nuxt/bridge` 导入的，不是从 `nuxt` 或 `nuxt/config`** ——这是 Bridge 项目与纯 3 项目在配置首行上最直观的差别。

而 **`bridge: false` 是刻意的初始状态**：下一步的说明写着"**If 'bridge' is set to false, your application will operate without any changes as before**"，也就是**先保证装完包、命令换完，应用行为完全不变**，再逐块打开。

## 第四步：命令改名为 nuxt2

> The `nuxt` command should now be changed to the `nuxt2` command.

三处脚本一起改：

```diff
{
  "scripts": {
-   "dev": "nuxt",
+   "dev": "nuxt2",
-   "build": "nuxt build",
+   "build": "nuxt2 build",
-   "start": "nuxt start",
+   "start": "nuxt2 start"
  }
}
```

改完立刻试跑一次 `nuxt2`，官方预期是"**application works as before**"。**这一步是安全网**：`bridge: false` ＋ 命令改名之后如果行为变了，说明问题出在依赖升级本身，而不是 Bridge 的任何特性。

## 八块可以分开迁的能力

`## Upgrade Steps` 先给一句总原则：**"The below `Upgrade Steps` does not need to be done all at once."** 然后按顺序列了**八个**入口：

TypeScript、Migrate Legacy Composition API、Plugins and Middleware、Migrate New Composition API、Meta Tags、Runtime Config、Nitro、Vite。

这八条就是 Bridge 剩余文档的目录，也是它"渐进"的具体含义：**每一块单独打开、单独验证**。第三、第四两条都动到中间件与插件，所以中间件那一页的 `defineNuxtRouteMiddleware` 与 `definePageMeta` 各自属于不同阶段，别混着做。

## Migrate from CommonJS to ESM

最后一个小节与模块作者那一页给的是同一句话：**"Nuxt 3 natively supports TypeScript and ECMAScript Modules. Please check Native ES Modules for more info and upgrading."** ——**具体迁移动作不在这两页里**，只给指路。

## 自查

- 定位：前向兼容层，靠装一个模块体验多数 3 特性；官方用词是 "(almost) ready"。
- 两条前置：新项目**不要**用 Bridge；**`useAsyncData` 与 `useFetch` 不可用**。
- 第一步顺序：**停 dev → 删锁文件 → 升到 2.17.x → 重装 → dev 与 build 都验过**。
- 装的是 `@nuxt/bridge` ＋ `nuxi`，**都是 dev 依赖**；配置从 `@nuxt/bridge` 导入 `defineNuxtConfig`，**初始 `bridge: false`**。
- 脚本里 `nuxt` → `nuxt2` 三处，改完先确认行为不变再开特性。
- 八块能力可分开迁，不必一次做完；ESM 迁移只给了指路链接。

## 下一篇

下一篇 618 读 Bridge 的配置页：**十二个特性开关分两组**——四个是"要显式打开"的（`vite`、`meta`、`macros.pageMeta`、`typescript.esbuild`），七个是"默认就开着、可以关掉"的（`nitro`、`app`、`capi`（含 `capi.legacy` 这种细粒度写法）、`transpile`、`scriptSetup`、`imports`、`constraints`）；官方建议是**"always best to stick with defaults!"**，而这一页还顺带给出**两条旧选项的确切去向：`router.base` → `app.baseURL`、`build.publicPath` → `app.cdnURL`**。

## 本文依据

- https://nuxt.com/docs/4.x/bridge/overview（`::note` 新项目跳过句、`::warning` 关于 `useAsyncData`／`useFetch` 不可用句、forward-compatibility layer 定义句与 "(almost) ready"／"gradually proceed" 句；Upgrade Nuxt 2 四个动作（停 dev、删 `package-lock.json` 与 `yarn.lock`、`^2.16.3`→`^2.17.3` diff、五种重装写法）与其 `::note` 检查点；Install Nuxt Bridge 句与五种 `-D` 写法；Update `nuxt.config` 的 CommonJS 警告句、静态/动态 import 与改名 `.ts` 建议、`bridge: false` 示例；Update Commands 三处 diff、"works as before" 与 "(If 'bridge' is set to false…)" 括注；Upgrade Steps 的"不必一次做完"句与八个入口清单；Migrate from CommonJS to ESM 一句）
