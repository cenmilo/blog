---
url: /posts/be2f1603cbcc659d01454040f9748b67/
title: hooks：冒号命名与层级对象，示例只演示了一层嵌套
date: 2027-04-25T10:00:00+08:00
updated: 2027-04-25T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## hooks` 有两句定性与一个示例，**没有类型行、没有默认值行、没有 See**。两句说的是：钩子是 Nuxt 事件的监听器，**通常在模块里用，但在 `nuxt.config` 里也可用**；内部命名用冒号（例如 `build:done`），**为了配置方便，可以在 `nuxt.config` 里把它们组织成层级对象**。示例把这条规则演示成 `hooks: { build: { done (builder) { … } } }`——在构建目录里写了一个额外文件。**关键空白是：钩子清单里存在两级冒号的名字（`nitro:build:before`、`app:data:refresh`、`page:loading:start`），但官方示例只演示了一级嵌套，能不能写成两级对象、两种写法能否混用，本节没说。**

categories:
  - 前端开发

tags:
  - nuxt4
  - 钩子
  - 构建期事件
  - 命名约定
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

想在构建产物落盘之前往 `.nuxt` 里塞一个文件，最省事的做法是在 `nuxt.config` 里直接挂一个钩子——不用为这一件事写模块。这一节讲的就是这个入口。

## 这一节的两句定性

> Hooks are listeners to Nuxt events that are typically used in modules, but are also available in `nuxt.config`.
>
> Internally, hooks follow a naming pattern using colons (e.g., build:done).
> For ease of configuration, you can also structure them as an hierarchical object in `nuxt.config` (as below).

**没有 `**Type**`、没有 `**Default**`、没有 See。** 第二句里的 "as below" 指的正是下面那唯一一个示例。

## 唯一的示例

```ts [nuxt.config.ts]
import fs from 'node:fs'
import path from 'node:path'

export default defineNuxtConfig({
  hooks: {
    build: {
      done (builder) {
        const extraFilePath = path.join(
          builder.nuxt.options.buildDir,
          'extra-file',
        )
        fs.writeFileSync(extraFilePath, 'Something extra')
      },
    },
  },
})
```

从这段能读出的事实，比定性句还多三条：

1. 冒号名 `build:done` 被写成了 `build: { done (…) }`——**这就是"层级对象"的含义：按冒号切开，做成嵌套键**。
2. 回调的参数在这里叫 `builder`，并且示例从它身上取 `builder.nuxt.options.buildDir` 作为落盘位置——**要往构建目录写东西，官方示范的路径就是从 builder 上取 `nuxt.options.buildDir`**。
3. 示例是**同步函数**，没有 `await`，也没有返回值——但这是示例的选择，**本节没有规定钩子回调能不能是 async、有没有返回值约定**。

## 两级冒号的名字怎么办：本节没答

钩子清单在 `api/advanced/hooks` 那一页，分三张表：**App Hooks（runtime）**、**Nuxt Hooks（build time）**、**Nitro App Hooks（runtime, server-side）**。构建期那张表就是"能在 `nuxt.config` 里挂"的那一批，而官方给完整清单的方式是指向源码：

- App Hooks 那张表前写着 "Check the [app source code]" 去找所有可用钩子（指 `packages/nuxt/src/app/nuxt.ts`）。
- Build Time 那张表前写着 "Check the [schema source code]"（指 `packages/schema/src/types/hooks.ts`）。
- Nitro 那张表让你去 Nitro 的 plugins 文档看全部钩子。

问题在这里：这些名字**不只有一个冒号**。清单里出现的两级命名包括 `nitro:build:before`、`nitro:build:public-assets`、`app:data:refresh`、`app:error:cleared`、`app:manifest:update`、`app:suspense:resolve`、`page:loading:start`、`page:loading:end`。

而配置参考的唯一示例只演示了**一级**（`build:done`）。所以：

- `nitro:build:before` 能不能写成 `hooks: { nitro: { build: { before } } }` ——**没有记载**。
- 能不能直接写字符串键 `'nitro:build:before' (nitro) { … }` ——本节也没示范（这种冒号字符串键的写法只在 `nitro.hooks` 里出现过，那是另一个键，见下）。
- 两种写法能不能在同一个 `hooks` 对象里混用 ——**无记载**。

要落地，只能两种都试一次，看构建日志里钩子有没有被调用。

## 与 `nitro.hooks` 的分界

`nitro` 那一节下还有一处钩子入口：`nitro: { hooks: { 'prerender:generate' (route) { … } } }`（预渲染页生成时按路由跳过某页的写法）。两者不是一回事：

| | 顶层 `hooks` | `nitro.hooks` |
| --- | --- | --- |
| 事件来源 | Nuxt 自身的构建期事件 | Nitro 的事件 |
| 文档给的写法 | **层级对象**（示例） | **冒号字符串键**（示例） |
| 完整清单 | `api/advanced/hooks` 的 Build Time 表 ＋ 指向 schema 源码 | 指向 Nitro 文档 |

也就是说，文档自己在两处用了**两种形状**来写钩子键名。这恰好说明"为什么顶层示例选层级对象"是排版选择而非规则——但**这个推论没有写在文档里**，本节也没说两者可以互换。

## 自查

- `## hooks`：两句定性 ＋ 一个示例，**无 Type、无 Default、无 See**。
- 层级对象的含义是"按冒号切键"，示例给的是 `build:done` → `build: { done (builder) }`。
- 官方示例还示范了从 `builder.nuxt.options.buildDir` 取落盘位置、以及同步回调的写法；**async/返回值规则未写**。
- 清单里存在两级冒号名（`nitro:build:before`、`app:data:refresh`、`page:loading:start` 等八个），**本节没有演示两级对象怎么写**。
- 完整钩子名以源码为准：App 表指 `packages/nuxt/src/app/nuxt.ts`、Build Time 表指 `packages/schema/src/types/hooks.ts`、Nitro 表指 Nitro 文档。
- `nitro.hooks` 的示例用的是冒号字符串键——**同一份文档里两种键名形状并存，能否互换未说明**。

## 下一篇

下一篇 587 看 `## devServer`：这一节有六个子项，但**记载完整度差别极大**——`port` 是唯一给全的（`number` ＋ `3000`），`host` 只有一句 "Dev server listening host" 连类型都没有，`cors.origin` 的默认值是一个空对象组成的数组 `[{}]`，`loadingTemplate` 写着 `function` 却没有参数说明，而 `url` 一边给了默认值 `"http://localhost:3000"` 一边写着**"不应该直接设置它"**。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## hooks` 两句定性与唯一示例全文；`## nitro` 下 `nitro.hooks` 的 `'prerender:generate'` 冒号键写法）
- https://nuxt.com/docs/4.x/api/advanced/hooks（三张表的划分 App Runtime / Nuxt Build Time / Nitro App Runtime、两条 "Check the … source code" 指向与源码路径、两级冒号钩子名 `nitro:build:before`、`nitro:build:public-assets`、`app:data:refresh`、`app:error:cleared`、`app:manifest:update`、`app:suspense:resolve`、`page:loading:start`、`page:loading:end`）
