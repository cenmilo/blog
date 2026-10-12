---
url: /posts/241d479de7e9ff47887e4e5e908c67e7/
title: Bridge 下的 Nitro 与 Vite：脚本换命令与那条 nitro 前置
date: 2027-05-07T11:00:00+08:00
updated: 2027-05-07T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这两页合起来是 Bridge 的"换服务端与换打包器"，但**它们的开关不是对等的**：Vite 那页开头一条 `::warning` 写得很硬——**"When using `vite`, nitro must have been configured."** 而 Nitro 这页的动作最多：删 `@nuxt/nitro`（Bridge 已注入同功能）、设 `bridge.nitro: true`、**装 `nuxi` 为 dev 依赖**、把脚本换成 `nuxi dev`／`nuxi build`／**`nuxi preview`**（一条 `::tip` 补了反面："If `nitro: false`, use the `nuxt2` command."）、按 `target: 'static'` 与否选 `nuxi generate` 还是 `nuxi build`、**把 `.output` 加进 `.gitignore`**，最后一节是验证：**用 `nuxi dev` 与 `nuxi build`（或 `nuxi generate`）试一遍**。Vite 那页更短：删 `nuxt-vite`、设 `bridge.vite: true` ＋ `bridge.nitro: true`，Vite 本身的配置写在**顶层 `vite`** 键里。

categories:
  - 前端开发

tags:
  - nuxt4
  - bridge
  - nitro
  - vite
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

Bridge 项目里改服务端与改打包器是两件事，但官方把顺序钉死了：先 Nitro，后 Vite。这一页还有一处容易漏——`npm start` 那条脚本换的不是 `nuxi start`，而是 **`nuxi preview`**。

## Nitro：六个动作

`bridge/nitro` 的六个 `##` 小节按顺序就是操作顺序。

**一、删包**：Remove `@nuxt/nitro`: Bridge injects same functionality.

**二、开关注**：

```ts [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  bridge: {
    nitro: true,
  },
})
```

**三、改脚本的理由**：

> You will also need to update your scripts within your `package.json` to reflect the fact that **Nuxt will now produce a Nitro server as build output.**

**四、装 nuxi**：`nuxi` 要作为 dev 依赖装上，五种包管理器写法都给全（`npm install -D nuxi` 起步，deno 是 `deno add -D npm:nuxi`）。

**五、脚本改名**：官方说 Nuxt 3 引入了新 CLI `nuxi`，改法是三行一起换：

```diff
{
  "scripts": {
-   "dev": "nuxt",
+   "dev": "nuxi dev",
-   "build": "nuxt build",
+   "build": "nuxi build",
-   "start": "nuxt start",
+   "start": "nuxi preview"
  }
}
```

**注意第三条不是 `nuxi start`**：`nuxt start` 对应的是 **`nuxi preview`**。紧跟的 `::tip` 给了反面：**"If `nitro: false`, use the `nuxt2` command."** ——这两句合起来才是完整规则：**开了 Nitro 走 `nuxi *`（start 用 preview），没开就整组留在 `nuxt2`**。这与前面 overview 那页"命令从 `nuxt` 改成 `nuxt2`"并不矛盾：那一步是在**还没开 Nitro** 的状态下做的。

**六、按 target 选构建命令**：

- `### Static Target`：如果你设过 `target: 'static'`，**要把 build 脚本改成 `nuxi generate`**。
- `### Server Target`：其它情况用 `nuxi build`（示例里同时给了 `"start": "nuxi preview"`）。

`target` 是 Nuxt 2 的键；4.x 侧对应的是"预渲染/静态产物"那套（`nuxt generate` 与 `nuxt build --prerender`），**Bridge 这一页没有把两边对齐讲**，只要求你据此换脚本。

## 收尾两节：gitignore 与验证

> ## Exclude Built Nitro Folder From Git
> Add the folder `.output` to the `.gitignore` file.

> ## Ensure Everything Goes Well
> ✔️ Try with `nuxi dev` and `nuxi build` (or `nuxi generate`) to see if everything goes well.

`.output` 是 Nitro 的产物目录——**这一页给的排除动作就是加进 `.gitignore`**。而验证只有两个命令位（dev 与 build/generate），**没有列出"不正常"会是什么症状**，所以撞上问题要回前面那两页（Overview 的检查点、以及各能力页的开关）逐项对照。

## Vite：三条动作加一条前置

`bridge/vite` 只有四个 `##` 小节，但第一条是硬前置：

> ⚠️ When using `vite`, **nitro** must have been configured.

接着：删 `nuxt-vite`（Bridge enables same functionality）；开关注释意明确：

```ts [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  bridge: {
    vite: true,
    nitro: true,
  },
})
```

**示例里 `vite: true` 与 `nitro: true` 是成对出现的**，正是那条前置的落地写法。最后一节 `## Configuration` 只说 Vite 的配置写在**顶层 `vite`** 键里：

```ts [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  vite: {
    // Config for Vite
  },
})
```

这与 Bridge 配置那页的分工一致：`bridge.vite` 决定"用不用 Vite"，`vite` 决定"Vite 怎么配"。

## 自查

- Nitro 六步：删 `@nuxt/nitro` → `bridge.nitro: true` → 改脚本的理由 → 装 `nuxi`（dev）→ 三条脚本（**start 换 `nuxi preview`**）→ 按 `target: 'static'` 选 `nuxi generate`／`nuxi build`。
- 反面规则记住：`nitro: false` 时整组脚本仍用 `nuxt2`。
- **`.output` 要进 `.gitignore`**；验证是 `nuxi dev` ＋ `nuxi build`（或 `nuxi generate`）。
- Vite 三条：删 `nuxt-vite`、`vite: true` **且 `nitro: true`**、具体配置写顶层 `vite`。
- **顺序不是可选的**：Vite 页的 `::warning` 把 Nitro 设为前置。

## 下一篇

下一篇 624 读 `bridge/nuxt3-compatible-api`（页标题是 New Composition API）：它给的是**七个旧 composable 的去处**——`ssrRef`/`shallowSsrRef` 换成要显式给 key 的 `useState`、`ssrPromise` 与 `onGlobalSetup` 直接移除、`useStore` 变 `useNuxtApp().$store`、`useContext`/`withContext` 变 `useNuxtApp`（附带一个**不建议使用、且 Nuxt 3 里不会存在的 `nuxt2Context`**）、`wrapProperty` 给了替代实现、`useAsync`/`useFetch` 换成 **`useLazyAsyncData`/`useLazyFetch`**。

## 本文依据

- https://nuxt.com/docs/4.x/bridge/nitro（六个 `##` 小节全部原文：Remove Modules 句、`bridge.nitro: true` 示例、Update Your Scripts 的理由句、Install Nuxi 与五种写法、Nuxi 段的三条脚本 diff 与 "If `nitro: false`, use the `nuxt2` command." 的 `::tip`、Static Target 与 Server Target 两节及两段 package.json 示例、`.output` 进 `.gitignore`、"✔️ Try with `nuxi dev` and `nuxi build` (or `nuxi generate`)"）
- https://nuxt.com/docs/4.x/bridge/vite（`::warning` 的 nitro 前置句、Remove `nuxt-vite`、`bridge: { vite: true, nitro: true }` 示例、`## Configuration` 的顶层 `vite` 示例）
- https://nuxt.com/docs/4.x/bridge/overview（`nuxt`→`nuxt2` 三条脚本改动，用于对照本页的 `nuxi` 改动与 `nitro` 开关的关系）
- https://nuxt.com/docs/4.x/bridge/configuration（`vite` 与 `nitro` 是两个独立 opt-in/default 项、Vite 具体配置写顶层 `vite`）
