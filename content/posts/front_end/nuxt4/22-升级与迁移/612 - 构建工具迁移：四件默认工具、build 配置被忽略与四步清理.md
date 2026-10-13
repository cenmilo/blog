---
url: /posts/72710f7f6edce3a34e2727b0d1f52c80/
title: 612 - 构建工具迁移：四件默认工具、build 配置被忽略与四步清理
date: 2027-05-04T09:00:00+08:00
updated: 2027-05-04T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`migration/bundling` 这一节的标题实际写着 **Build Tooling**（slug 却是 `bundling`）。内容很短但有一句杀伤力大的话：**"For this reason, most of your previous `build` configuration in `nuxt.config` will now be **ignored**, including any custom babel configuration."** ——旧配置里 `build` 下面的东西大部分**不报错、只是不被读**。要配构建工具改用三个顶层键 **`vite`、`webpack`、`postcss`**。默认工具列了四件（Vite **或** webpack、Rollup、PostCSS、esbuild），Steps 是**四步纯删除**：移除 `@nuxt/typescript-build` 与 `@nuxt/typescript-runtime`、移除没用的 babel 依赖、移除显式的 core-js 依赖、把 `require` 改成 `import`。**这一节末尾还留着一行 `TODO: Enabling webpack builder`——想切到 webpack 的说明，那一节没有写。**

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建工具
  - babel移除
  - vite与webpack
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

升级之后发现配置里那一大段 `build: { transpile: [...], babel: {...}, extractCSS: true }` 全都没效果了，也没人报错。这一节解释原因就是这句：那些配置**被忽略了**。

## 四件默认工具，一句"旧的 build 会被忽略"

页首定性句："Learn how to migrate from Nuxt 2 to Nuxt 3 build tooling."。正文先列默认在用的构建工具，**四条**：

- Vite **或** webpack
- Rollup
- PostCSS
- esbuild

紧接着就是那句最要紧的话：

> For this reason, most of your previous `build` configuration in `nuxt.config` will now be ignored, including any custom babel configuration.

这句话有两层：**"most … will be ignored"**（大部分不再生效，且是**静默**的——没有任何一句说会警告你），以及**点名 babel 配置整块作废**（`include` 里那句 "including any custom babel configuration"）。

替代入口也给得很具体：

> If you need to configure any of Nuxt's build tools, you can do so in your `nuxt.config`, using the new **top-level `vite`, `webpack` and `postcss` keys**.

**注意措辞是 top-level**：这三样要写在 `nuxt.config` 的最外层，而不是嵌回 `build` 里。这与配置参考里那三节（`## vite` 十五个 `###` ＋ 十个 `####`、`## webpack` 十七个 `###` ＋ 二十五个 `####`、`## postcss` 两个 `###` ＋ 两个 `####`）正好对上。

最后一句是 "In addition, Nuxt ships with TypeScript support." ＋ 一条指向 TypeScript 概念页的 read-more——**TypeScript 是内置的，这是下面第一步能删掉两个包的前提。**

## Steps：四步全是删除

这一节只有一个 `## Steps`，**四步**，性质都是"清理"：

1. 从**依赖与 modules 两处**移除 `@nuxt/typescript-build` 和 `@nuxt/typescript-runtime`。
2. 移除项目里任何不再使用的 babel 依赖。
3. 移除任何显式声明的 core-js 依赖。
4. 把 `require` 迁移到 `import`。

四步里有三点要展开：

- **第 1 步要求删两个地方**：`package.json` 的依赖，以及 `nuxt.config` 里 `modules` 数组中的这两项。很多人只删了依赖没删 modules，或反之。
- **第 2、3 步是"删依赖"而不是"改配置"**：babel 与 core-js 的 polyfill 逻辑都不再由你接管，这一步留着不改，只是包体积与安装时间变大，**并不会报错**。
- **第 4 步与 auto-imports 那一节的 ESM 规则是同一件事**：`require` 换成 `import`，因为整个配置链按 ESM 走。

## 这一节没写完的那一段

源码里在、页面上不渲染的一行注释：**`<!-- TODO: Enabling webpack builder -->`**。

也就是说：**"怎么把构建器从 Vite 换成 webpack"这一段，这一节是空的**。而这个能力确实存在——配置参考的 `## builder` 那节给出了三种写法（含 `"webpack"` 这种简写，并且官方写明 webpack／rspack 需要你自己安装对应包），`## server` 那节也有服务端构建器的选项。

**所以要用 webpack 时，这一节帮不上忙，得去配置参考与构建器那一节找。** 这是这一节目前最大的缺口。

## 自查

- 默认工具四件：**Vite 或 webpack、Rollup、PostCSS、esbuild**。
- 核心后果句：**旧 `build` 配置大部分被忽略（静默），自定义 babel 配置整块作废**。
- 新入口是**三个顶层键**：`vite`、`webpack`、`postcss`（不要嵌回 `build`）。
- 四步全是删除／替换：两个 TypeScript 旧包（依赖＋modules 两处）、babel 依赖、core-js 显式依赖、`require`→`import`。
- **切 webpack 的说明这一节没写**（TODO 未补），要去看 `## builder` 与构建器那一节。
- 页面标题 **Build Tooling** 与 slug `bundling` 不一致，检索时两个词都要试。

## 下一篇

下一篇 613 读 `migration/server`：迁移文档里剩下的两节，一节讲服务端侧的改动（下一篇），另一节讲模块作者要知道的兼容事项——605 那一节的 tip 把模块作者引向的正是后者。

## 本文依据

- https://nuxt.com/docs/4.x/migration/bundling（页标题 "Build Tooling" 与 slug `bundling`；定性句；四件默认工具列表；"For this reason, most of your previous `build` configuration … will now be ignored, including any custom babel configuration."；"…using the new top-level `vite`, `webpack` and `postcss` keys."；"In addition, Nuxt ships with TypeScript support." 与指向 TypeScript 概念页的 read-more；`## Steps` 四步原文；末尾 `<!-- TODO: Enabling webpack builder -->` 注释）
- https://nuxt.com/docs/4.x/api/nuxt-config（`## builder` 的三种写法与 webpack／rspack 需自行安装、`## server` 的服务端构建器、`## vite`／`## webpack`／`## postcss` 三节的子项数量，用于印证"三个顶层键"的落点）
