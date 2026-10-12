---
url: /posts/33f756e7e2896884bf7277f6b8e687ef/
title: 构建期钩子四十五到五十二：webpack 八行与那三条 WebpackBar 事件
date: 2027-05-24T10:00:00+08:00
updated: 2027-05-24T10:00:00+08:00
author: cmdragon

summary:
  摘要：构建期表的最后八行全是 webpack，**四行讲配置、两行讲编译、四行讲 WebpackBar**：**`webpack:config`（在配置 webpack 编译器之前）、`webpack:configResolved`（读取已解析的 webpack 配置）、`webpack:compile`（紧接编译之前）、`webpack:compiled`（在资源被加载之后）**，然后 **`webpack:change`、`webpack:error`、`webpack:done`、`webpack:progress` 的描述都直接写着 WebpackBar 上的事件名（`change`、`done` 且有错误、`allDone`、`progress`）**。两处细节：前四行的参数是 **`webpackConfigs` 复数**（一份客户端一份服务端），这与 Vite 侧在 Nuxt 5 要改成"共享配置"正好相反；`webpack:compiled` 的描述**不是"编译完成"而是"资源被加载之后"**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建期钩子
  - webpack
  - WebpackBar
  - 编译时机
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

Vite 那五行之后，表里剩下八行都属于 webpack。这一组的价值不在数量，而在于**有四行的描述把触发点交给了另一个库的事件名**。

## 八行照录

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `webpack:config` | `webpackConfigs` | 在配置 webpack 编译器之前调用 |
| `webpack:configResolved` | `webpackConfigs` | 允许读取已解析的 webpack 配置 |
| `webpack:compile` | `options` | 紧接编译之前调用 |
| `webpack:compiled` | `options` | **在资源被加载之后调用** |
| `webpack:change` | `shortPath` | 在 WebpackBar 上的 `change` 时调用 |
| `webpack:error` | - | 在 WebpackBar 上 `done` 时若有错误则调用 |
| `webpack:done` | - | 在 WebpackBar 上的 `allDone` 时调用 |
| `webpack:progress` | `statesArray` | 在 WebpackBar 上的 `progress` 时调用 |

按用途分三组：**改配置两行、编译时机两行、进度与错误四行**。

## 前四行：配置是复数

`webpack:config` 与 `webpack:configResolved` 的参数都是 **`webpackConfigs`（复数）**，而描述分别是"在配置编译器之前"与"读取已解析的配置"。

**复数这件事本身就是信息**：webpack 侧同时持有客户端与服务端两份配置，所以这两行拿到的是**一个数组**。这与 Vite 侧形成明确对照：

| 侧 | 配置形态 | 弃用状态 |
| --- | --- | --- |
| Vite | `vite:extendConfig`、`vite:configResolved` 参数是 `viteInlineConfig, env`——**靠 `env` 区分两侧** | **两行都标 "Deprecated in Nuxt 5+"，理由是 Nuxt 5 用共享配置** |
| webpack | `webpackConfigs` 数组 | **没有任何弃用标记** |

所以"按端分别配置"这个模型在 webpack 侧仍然是通过**数组元素**表达的，而它正在从 Vite 侧退场。

`webpack:config` 的时机是"配置编译器之前"，**这是能改配置的最后位置**；`webpack:configResolved` 描述用的是 "Allows reading"——**只说读，没说改**。

## 中间两行：`compiled` 不等于编译完成

`webpack:compile` 与 `webpack:compiled` 的参数都写作 `options`，但描述不一样：

| 钩子 | 描述 |
| --- | --- |
| `webpack:compile` | 紧接编译之前调用 |
| `webpack:compiled` | **在资源被加载之后调用**（"Called after resources are loaded."） |

**第二行不要按名字理解成"编译全部结束"**——文档给的触发条件是"资源加载之后"。**这与构建器那一章里"`nuxt.server.reload()` 由编译器的 compiled 事件驱动"是同一个时点**：那一句写的是"官方构建器会发出一个 `compiled` 钩子（例如 `vite:compiled`、`webpack:compiled`）供服务端集成监听以触发重载"。

也就是说，**这一行是热重载的触发源之一**，而不只是观测点。

## 后四行：描述里带着别人的事件名

这四行的描述结构相同——**"Called on `<事件>` on WebpackBar"**，也就是**它们的触发点由 WebpackBar 这个库决定**：

| 钩子 | WebpackBar 侧 | 参数 |
| --- | --- | --- |
| `webpack:change` | `change` | `shortPath` |
| `webpack:error` | `done`，且**有错误时** | 无 |
| `webpack:done` | `allDone` | 无 |
| `webpack:progress` | `progress` | `statesArray` |

三点值得单独记：

**第一，`webpack:error` 的触发条件是 "`done` if has errors"**——它不是"发生错误时"，而是"收尾时发现有过错误"。**构建期真正发生错误的通知在另一行**：`build:error`（构建期表第 33 行，参数 `error`）。两者别混用。

**第二，`webpack:change` 的参数是 `shortPath` 而不是 `path`**。构建期表里另一行 `builder:watch` 的参数写作 `event, path`，而那里的 `path` 是相对项目根的路径（官方示例用 `resolve(nuxt.options.srcDir, relativePath)` 还原绝对路径）。**`shortPath` 是 WebpackBar 的短显示路径，两者不能假设同值**——**页面没有写出 `shortPath` 的确切形状**。

**第三，`webpack:done` 对应的是 `allDone`**，这个命名说明 webpack 会分多次 `done`（客户端、服务端），**全部完成才叫 `allDone`**。表里没有 `webpack:done` 单次的对应钩子，**想区分"某一侧完成"与"全部完成"，这一组只给了后者**。

## 与 Vite 五行的关系

Vite 那一族的五行依次是 `vite:extend`、`vite:extendConfig`、`vite:configResolved`、`vite:serverCreated`、`vite:compiled`；webpack 这一族是上面这八行。**表里并没有把它们一一对应**，能确定地说出来的只有三点：

| 差别 | Vite 五行 | webpack 八行 |
| --- | --- | --- |
| 配置形态 | `viteInlineConfig, env`，**靠 `env` 区分两侧** | **`webpackConfigs` 数组**，两侧各一份 |
| 弃用情况 | `vite:extendConfig` 与 `vite:configResolved` **两行标 "Deprecated in Nuxt 5+"**（理由是 Nuxt 5 改用共享配置） | **没有任何一行带弃用标记** |
| 进度类钩子 | **五行里没有进度类** | **四行都是进度与错误类**（都指向 WebpackBar 的事件） |

**"按端分别配置"这个模型正在从 Vite 侧退场，而 webpack 侧仍用数组表达它**——这是两组之间最实在的一条区别。**页面没有解释为什么 webpack 侧不带弃用标记**，只给出了行本身。

## 用哪一行

| 你要做的 | 挂哪一行 |
| --- | --- |
| 给 webpack 加 loader/plugin 前的最后改写 | `webpack:config` |
| 只是想看最终配置 | `webpack:configResolved` |
| 驱动进度条或终端状态 | `webpack:progress` ＋ `webpack:change` ＋ `webpack:done` |
| 判断"这次构建到底失败了没有" | `webpack:error`（**发现过错误才触发**），或 `build:error` |
| 在产物可用后做事 | `webpack:compiled`（**注意语义是资源加载之后**） |

这一组未记载的还有：`webpack:config` 被调用几次（客户端与服务端各一次还是合并一次）、`options` 在 `compile`／`compiled` 两行里分别有什么字段、`statesArray` 的形状、以及这些钩子在换用 Rspack 时是否存在对应行（**表里没有任何 rspack 钩子**）。

下一篇读第三张表：Nitro 运行时那九行。

## 本文依据

- https://nuxt.com/docs/4.x/api/advanced/hooks
