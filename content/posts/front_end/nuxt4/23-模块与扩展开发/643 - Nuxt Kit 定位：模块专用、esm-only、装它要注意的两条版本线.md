---
url: /posts/e43f9782b663c6cd70863d046293debb/
title: Nuxt Kit 定位：模块专用、esm-only、装它要注意的两条版本线
date: 2027-05-14T10:00:00+08:00
updated: 2027-05-14T10:00:00+08:00
author: cmdragon

summary:
  摘要：这一页只讲三件事，但每一条都会决定你的模块能不能加载。**第一，装法**：把 `@nuxt/kit` 写进 `package.json` 的 `dependencies`，而且**即使 Nuxt 已经间接装了它，官方也建议显式安装**；配套那条 `::note` 是版本下限——**`@nuxt/kit` 与 `@nuxt/schema` 单独安装时，版本必须大于等于你的 `nuxt` 版本**。**第二，作用域**：Kit 的工具**只给模块用，不要在设计运行时代码时导入它**（组件、组合式函数、页面、插件、服务端路由都不行）。**第三，包形态**：Kit 是 **esm-only** 的，**`require` 拿不到**，CommonJS 里唯一的绕法是 `await import()`。理解这三条之后，才谈得上按类别去查那二十来页 API。

categories:
  - 前端开发

tags:
  - nuxt4
  - NuxtKit
  - 模块作者
  - ESM
  - 依赖管理
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块里那行 `import { addPlugin, createResolver } from '@nuxt/kit'` 你大概抄过很多次。这一页回答的是抄之前该知道的三件事：**它装在哪个依赖段、什么代码里能用、以及为什么 `require` 会失败**。

## Kit 是干什么的

页标题下面那句话就是定位：**`@nuxt/kit` 为模块作者提供功能**。正文再具体一层：它提供的是一组**可组合的工具**，用途是**与 Nuxt 钩子、Nuxt 接口以及开发 Nuxt 模块**这三件事打交道，让这些都变得简单。

三个用途里前两个是第一次见的名词：**钩子**是框架在启动流程固定点留出的回调位置（**Nuxt 接口**指的是那个 `nuxt` 实例本身，内部机制那一页有定义）。也就是说 Kit 不是"另一套运行时"，**它是把配置与钩子这两件事包成函数的壳**。

## 装它的两条版本线

安装说明有两处限定，都不显眼：

> You can install the latest Nuxt Kit by adding it to the `dependencies` section of your `package.json`. However, please consider always explicitly installing the `@nuxt/kit` package even if it is already installed by Nuxt.

**要点是"显式安装"**：Nuxt 自己会带来 Kit，但那不是你依赖它的理由。

`::note` 给的是版本关系：

> `@nuxt/kit` and `@nuxt/schema` are key dependencies for Nuxt. If you are installing it separately, make sure that the versions of `@nuxt/kit` and `@nuxt/schema` are equal to or greater than your `nuxt` version to avoid any unexpected behavior.

这里把 **`@nuxt/schema` 也算进来**，方向是"两个包都要 **≥ 你的 nuxt 版本**"。

**低于就会出问题**，而官方对"出什么问题"只写了 "unexpected behavior" 一句，没给具体症状。所以排查时不要把 Kit 的报错当成唯一线索——**某个 Kit 函数在你的项目里不存在，可能只是版本装低了**。

页面给的示例形态值得注意，它写的是 nightly 别名：

```json [package.json]
{
  "dependencies": {
    "@nuxt/kit": "npm:@nuxt/kit-nightly@latest"
  }
}
```

**这是"提前试未发布版本"那条渠道在 Kit 上的写法**（`npm:` 前缀把 `@nuxt/kit-nightly` 这个包重命名成 `@nuxt/kit` 来装），普通项目按版本号写就行。

导入只有一个动作：

```ts [test.mjs]
import { useNuxt } from '@nuxt/kit'
```

## 只在模块可用，运行时一律不行

一条 `::note` 划定了作用域：

> Nuxt Kit utilities are only available for modules and not meant to be imported in runtime (components, Vue composables, pages, plugins, or server routes).

括号里点名了五种位置：**组件、Vue 组合式函数、页面、插件、服务端路由**。这解释了为什么把 Kit 函数写进 `app/plugins/` 里某个文件时会报解析错误——**Kit 属于构建期，运行时的产物里根本没有它**。

这条与前面模块结构页那条警告是同一族：**发布的模块在自己的 runtime 目录里连自动导入都用不了**，两边都是"运行时那一侧没有构建期的东西"。

## esm-only：`require` 直接不可用

原句是：Nuxt Kit 是一个 **esm-only 包**，这意味着你**不能** `require('@nuxt/kit')`。

CommonJS 里的绕法用动态导入：

```ts [test.cjs]
// This does NOT work!
// const kit = require('@nuxt/kit')
async function main () {
  const kit = await import('@nuxt/kit')
}
main()
```

注释把失败写法原样留在代码里（`// This does NOT work!`），这是本系列里少数"文档直接把错误写法贴出来"的地方。**含义是同步路径不存在**：你拿不到一个能立即使用的 `kit` 对象，只能在 `await` 之后用。

## 按类别去找函数

Kit 的 API 分成二十来页，一页管一类。这一页只写了怎么用，没有目录——下表是各页的函数数量，用来决定"这件事该翻哪页"：

| Kit 的 API 页 | 页上的函数个数 |
| --- | --- |
| nitro | 9 |
| compatibility | 7 |
| builder | 6 |
| programmatic | 5 |
| resolving | 4 |
| templates | 4 |
| autoimports | 3 |
| pages | 3 |
| components | 2 |
| context | 2 |
| dependencies | 2 |
| logging | 2 |
| modules | 2 |
| plugins | 2 |
| runtime-config | 2 |
| app-config | 1 |
| head | 1 |
| layers | 1 |
| layout | 1 |
| examples | 0（整页是示例集合，没有函数条目） |

三个查法上的提示：

**服务端相关的工具全在 nitro 那一页**（数量最多，9 个），模块要往服务端注入东西时先翻它，而不是在 pages 或 plugins 页里找。

**注入资产的动作分散在五页**：autoimports（组合式函数与工具）、components（组件）、pages（路由与路由规则）、plugins（插件）、layout 与 head（布局与全局 head）。**没有"一个函数管所有注入"这种入口**。

**examples 页没有函数**——它给的是完整模块的例子，遇到"我知道要做什么但不知道顺序"时从那页入手比翻单函数页更快。

## 上手时的三个检查

1. `package.json` 里 `@nuxt/kit` 是**显式写在 `dependencies`** 还是只靠 Nuxt 带进来？
2. `@nuxt/kit` 与 `@nuxt/schema` 的版本**是否都不低于 `nuxt` 的版本**？
3. 有没有任何一处 Kit 导入出现在 `app/` 或 `runtime/` 里的文件中——**那种位置一律不该出现**？

下一篇讲 modules 那一页，它只有两个函数，其中一个已经被废弃。

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/kit
- https://nuxt.com/docs/4.x/api/kit/modules
