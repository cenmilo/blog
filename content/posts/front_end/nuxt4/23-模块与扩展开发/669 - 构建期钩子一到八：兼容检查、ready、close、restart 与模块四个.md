---
url: /posts/986288266d26f0d6a3d01f10a08d88e6/
title: 构建期钩子一到八：兼容检查、ready、close、restart 与模块四个
date: 2027-05-23T09:00:00+08:00
updated: 2027-05-23T09:00:00+08:00
author: cmdragon

summary:
  摘要：构建期那张表有 52 行，头八行是**实例与模块的生命周期**：`kit:compatibility`（参数 `compatibility, issues`，作用是"允许扩展兼容性检查"）、**`ready`（Nuxt 初始化完成、实例可用）**、**`close`（实例正在优雅关闭）**、**`restart`（带 `{ hard? }`，是"应当被调用来重启当前实例"的那个**——注意它的描述是"要被调用"，不是"发生时调用"）、`modules:before` 与 `modules:done`（**在装用户模块之前／之后**，成对、无参数），以及 `module:before` 与 `module:done`（**每个模块各自的前后**，参数是 `module`）。**`module:done` 的描述里有一句很实用的信息：它带着"这个模块花了多久"**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建期钩子
  - 模块生命周期
  - 实例生命周期
  - 兼容性检查
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块作者最先要知道的钩子都在这几行里：**什么时候实例算准备好了、什么时候该收尾、以及"所有模块都装完了"是哪一个点**。这一组八个还包含一个容易读反的（`restart`）。

## 八行照录

这张表**只有三列：Hook、Arguments、Description**，**没有 Environment**（构建期只发生在 Node 侧）。

| 钩子 | 参数 | 描述 |
| --- | --- | --- |
| `kit:compatibility` | `compatibility, issues` | 允许扩展兼容性检查 |
| `ready` | `nuxt` | 在 Nuxt 初始化之后调用，此时 Nuxt 实例已经可以使用 |
| `close` | `nuxt` | 当 Nuxt 实例正在优雅关闭时调用 |
| `restart` | `{ hard?: boolean }` | **应当被调用来重启当前 Nuxt 实例** |
| `modules:before` | - | 在 Nuxt 初始化期间、**安装用户模块之前**调用 |
| `modules:done` | - | 在 Nuxt 初始化期间、**安装用户模块之后**调用 |
| `module:before` | `module` | 在每一个单独的模块被 setup 之前立刻调用 |
| `module:done` | `module` | 在每一个单独的模块被 setup 之后立刻调用，**包括它花了多长时间** |

## `restart` 的读法要反过来

五行里只有 `restart` 的描述不是"……时调用"，而是 **"To be called to restart the current Nuxt instance."**

**它是让你调用的，不是通知你的**：语义是"调用它来重启当前实例"，参数 `{ hard?: boolean }` 决定是不是硬重启。

**`hard` 为真与为假各自做什么、和 `nuxt.close()` ＋ 重新 `loadNuxt()` 有什么区别，表里没有解释**。这一行和另外七行不属于同一类，读到就得分开记。

顺带一提，`{ hard?: boolean }` 是这张表里少见的**带类型的参数写法**（其他多数只写名字，像 `nuxt`、`module`）。

## `ready` 与 `close`：一对开合

**`ready`**：Nuxt 初始化之后调用，实例可用。

**`close`**：实例正在优雅关闭时调用。

这两行是其他页反复引用的锚点，三处已经见过的用法正好落在这里：

| 用法 | 出处 |
| --- | --- |
| **`useNitro()` 只能在 `ready` 之后调用** | Kit 的 Nitro 那一页的 `::warning` |
| **模块打开了、持有了或启动了 watcher，应在 Nuxt 生命周期结束时关掉——用 `close`** | 模块高级配方那条 Module cleanup |
| 配置里 `hooks: { close: () => {} }` 与模块里 `nuxt.hook('close', ...)` | 钩子用法页给的两个最小示例 |

第一条最容易踩：**在 `setup` 里直接调 `useNitro()` 是不行的**，要放进 `ready` 的回调。参数就一个 `nuxt` 实例，**与另外两条 `nuxt` 参数一致**。

## `modules:` 与 `module:` 的分别

**四个模块相关钩子的名字只差一个 s，语义完全不同**：

| 钩子 | 范围 | 参数 | 出现次数 |
| --- | --- | --- | --- |
| `modules:before` | **整批**用户模块 | 无 | **一次** |
| `modules:done` | **整批**用户模块 | 无 | **一次** |
| `module:before` | **每一个**模块 | `module` | 每个模块一次 |
| `module:done` | **每一个**模块 | `module` | 每个模块一次 |

`modules:before`／`modules:done` 两句描述都带着"**在 Nuxt 初始化期间**"这个限定，所以它们发生在启动阶段而不是构建阶段。

**`modules:done` 是一个特殊的位置**：模块高级配方那一节明确用它作时机——**要让你自定义的钩子被别人订阅到，就在 `modules:done` 里 `callHook`**，理由是这时所有其他模块都已经有机会在自己的 `setup` 里注册监听。

也就是说：**`modules:done` 之后，模块之间的"谁还没装"这个问题已经没有悬念了**。

## `module:done` 里带耗时

这一行的描述是八行里唯一给出度量信息的：**每个单独模块 setup 之后立刻调用，包括它花了多久（including how long it took）**。

它与最佳实践那条阈值正好配成一组：**模块 setup 超过 1 秒，Nuxt 会发出警告**。

组合起来的用法很实际：**`module:done` 是"按模块看耗时"的位置，而那条 1 秒警告是框架自己给的门槛**。要做一个"哪个模块最慢"的观测，就在 `module:done` 里读那个耗时；**耗时字段的具体名字与结构，表里没写**（参数只写 `module`）。

## `kit:compatibility` 这行最容易被略过

描述一句：**允许扩展兼容性检查**，参数是 `compatibility, issues`（**两个参数依次传**，与 `err, target, info` 那种写法一致）。

它对应的是这一族机制的开放点：**兼容性检查本身由 `meta.compatibility` 驱动**（不满足时表现是控制台 WARN 并把模块禁用），而这个钩子允许**在框架已有的检查之外补充你自己的判断**——`issues` 就是那份可往里面加条目的问题列表。

**页面没有给出示例，也没说加进去的 issue 会怎样影响"模块被禁用"这个决定**，这两点属于文档未说明。

## 读这张表的两个习惯

**第一，别按行号推顺序。** 表里的排列大致成族（实例、模块、app、build、builder、pages、server、imports、components、nitro、schema、vite、webpack），**但除了少数几行明确写出先后关系之外，文档没有承诺"上一行一定早于下一行"**。这张表里说法明确的是：`modules:before`／`modules:done` 的"之前／之后"、`module:before`／`module:done` 的"立刻前／立刻后"，以及后面几行的 `pages:routerOptions`（**数组里靠后的覆盖靠前的**）、`components:dirs`（**在 `app:resolve` 之内调用**）这两类。

**第二，Arguments 列是形状而不是类型。** 只有个别行写了 TS 形态（像 `{ hard?: boolean }`），其余是名字或逗号分隔的名字列表。**要知道参数具体是什么，得去那一节开头给的 schema 源码链接**。

下一篇读九到二十行：`app` 一族、`build` 与 `builder` 一词之差、以及页面路由扫描的两段。

## 本文依据

- https://nuxt.com/docs/4.x/api/advanced/hooks
