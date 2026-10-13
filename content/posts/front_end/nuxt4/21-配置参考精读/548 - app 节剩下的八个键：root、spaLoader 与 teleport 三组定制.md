---
url: /posts/77a4d6857029a698271a34c4511a5277/
title: 548 - app 节剩下的八个键：root、spaLoader 与 teleport 三组定制
date: 2027-04-12T11:00:00+08:00
updated: 2027-04-12T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`app` 节剩下的八个键都在改"挂在 DOM 上的那几个容器元素"：`rootAttrs`（`object`，默认 `{"id": "__nuxt"}`）、`rootId`（`string`，默认 `"__nuxt"`）、`rootTag`（默认 `"div"`）、`spaLoaderAttrs`（默认 `{"id": "__nuxt-loader"}`，下面还多一个 `id` 子项）、`spaLoaderTag`（默认 `"div"`）、`teleportAttrs`（默认 `{"id": "teleports"}`）、`teleportId`（默认 `"teleports"`）、`teleportTag`（默认 `"div"`）。**这一节有一处明显的复制痕迹：`rootAttrs` 与 `rootId` 的说明文字一模一样，都写 "Customize Nuxt root element id."**，尽管前者是对象、后者是字符串。第三组要小心：`<Teleport>` 那一节写着 **Nuxt 目前只对 teleports 到 `#teleports` 提供 SSR 支持**，而 `#teleports` 正是 `teleportId` 的默认值。

categories:
  - 前端开发

tags:
  - nuxt4
  - 配置
  - teleport
  - spa
  - dom
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

页面源码里那个 `<div id="__nuxt">`、还有一处 `<div id="teleports">`，都不是你写的。

它们的标签名和 id 由三组配置项决定，这一篇把 `app` 节剩下的八个键读完。

## 三组、八行，全部照录

| 子项 | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `rootAttrs` | Customize Nuxt root element id. | `object` | `{"id": "__nuxt"}` |
| `rootId` | Customize Nuxt root element id. | `string` | `"__nuxt"` |
| `rootTag` | Customize Nuxt root element tag. | `string` | `"div"` |
| `spaLoaderAttrs` | Customize Nuxt SPA loading template element attributes. | `object` | `{"id": "__nuxt-loader"}` |
| `spaLoaderAttrs.id` | （该 `id` 子项没有说明句） | `string` | `"__nuxt-loader"` |
| `spaLoaderTag` | Customize Nuxt SpaLoader element tag. | `string` | `"div"` |
| `teleportAttrs` | Customize Nuxt Teleport element attributes. | `object` | `{"id": "teleports"}` |
| `teleportId` | Customize Nuxt Teleport element id. | `string` | `"teleports"` |
| `teleportTag` | Customize Nuxt Teleport element tag. | `string` | `"div"` |

表里八行是 `###` 级的键（`spaLoaderAttrs.id` 是它下面唯一的 `####` 子项）。三组的形状一致：**每组都是「Attrs（对象）＋ Id（字符串）＋ Tag（字符串）」，Tag 默认全是 `"div"`。** 只有 spaLoader 那组多出一个没有说明句的 `id` 子项。

## 一处复制痕迹，别按字面读

`rootAttrs` 与 `rootId` 的说明**完全相同**，都写着 "Customize Nuxt root element id."。但两者形态不同：一个 `object`（默认只有一个 `id` 键），一个 `string`。

**这说明 `rootAttrs` 那行的文字是从 `rootId` 复制来的**——它真正的含义要按默认值读：**给根元素设属性**（对象里可以放任意属性，官方默认只给了 `id`）。对照 `teleportAttrs` 那行的措辞就更清楚，那里写的是 "…element **attributes**"，`spaLoaderAttrs` 写的也是 "…element **attributes**"——**只有 `rootAttrs` 这行漏掉了 attributes 这个词**。

所以三组"Attrs"的读法是统一的：那是属性对象。官方在这一节没写"Attrs 与 Id 同时设会怎样"，从默认值的形状看两者重叠在同一键（`id`）上——**真要改，只改一组，别混着写**。

## 三个标签名能改成什么

`rootTag`／`spaLoaderTag`／`teleportTag` 都是 `string`、默认 `"div"`，**说明句只说"定制标签"，没有给可选范围**。

这一处的限制要诚实标注：**能换成哪些标签、换了对 CSS 与 SSR 有什么影响，文档没写**。语义上更合适的容器（比如 `main`）能不能直接当 `rootTag` 用，官方没有表态——真要动，验证办法是改完跑一次 `nuxt build`，看产物 HTML 的根元素与客户端挂载是否都正常。

## teleportId 这一项有配套约束

`<Teleport>` 组件那一节有一条 `::warning`，正好落在 `teleportId` 上：

> The `to` target of `<Teleport>` expects a CSS selector string or an actual DOM node. Nuxt currently has SSR support for teleports to `#teleports` only, with client-side support for other targets using a `<ClientOnly>` wrapper.

官方给的示例就是写到那个默认位置：

```vue
<template>
  <Teleport to="#teleports">
    <div v-if="open" class="modal">...</div>
  </Teleport>
</template>
```

三件事连起来读：

- **`#teleports` 这个选择器来自 `teleportId` 的默认值 `"teleports"`。**
- **只有这个目标的 teleport 有 SSR 支持**，其它目标只能靠 `<ClientOnly>` 在客户端渲染。
- 于是产生一个文档没回答的问题：**改了 `app.teleportId` 之后，SSR 支持跟着新 id 走，还是仍然只认字面的 `#teleports`？** 官方两处都没说。

这意味着改 `teleportId` 不是纯命名问题——它可能把服务端渲染的 teleport 变成只剩客户端。**除非确有冲突（宿主页里已经有同名元素），否则这一项保持默认更稳**；要改就实测一次带 `<Teleport>` 的页面在 SSR 下的输出（查看源码里 `teleports` 那个容器是否有内容，而不是只在开发者工具里看）。

## spaLoaderAttrs 管的是哪一段

`spaLoaderAttrs` 的说明写的是 "SPA **loading template** element attributes"，与 `## spaLoadingTemplate` 那一节呼应——`ssr: false` 时页面里那段占位内容的容器元素由它定制。

组合关系是：`spaLoaderTag` 决定用什么标签、`spaLoaderAttrs` 决定属性（默认 `id: "__nuxt-loader"`）。这一节没有指向 `spaLoadingTemplate` 的链接，两处并列写在同一份配置里，**要改加载占位的样式，位置在 `spaLoadingTemplate` 那节；改容器元素本身，在这两行**。

## 自查

- 三组形状相同：Attrs（对象）＋ Id（字符串）＋ Tag（字符串），Tag 默认都是 `"div"`；默认 id 分别是 `__nuxt`、`__nuxt-loader`、`teleports`。
- **`rootAttrs` 的说明句与 `rootId` 一字不差**（都写 "root element id"），按 `teleportAttrs`/`spaLoaderAttrs` 的措辞读，它其实是属性对象；Attrs 与 Id 重叠在 `id` 上，只改一组。
- 标签名的可选范围、换标签的影响文档未给，改后要构建验证。
- `teleportId` 有配套约束：`<Teleport>` 那一节说 SSR 只支持 `#teleports`；改 id 后 SSR 支持是否跟随，官方没写——保持默认最稳。
- `spaLoaderAttrs`/`spaLoaderTag` 改的是容器元素，占位内容本身在 `spaLoadingTemplate` 那节。

## 下一篇

下一篇 549 读 `css`：那个数组支持三种写法（直接给 npm 包名、给项目里的 CSS 文件路径、给预处理器文件路径），官方明确 Nuxt 按扩展名猜预处理器、但**该装的 loader 还得你自己装**，而且这一节没有默认值。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`### rootAttrs`/`### rootId`/`### rootTag`/`### spaLoaderAttrs`（含其 `#### id` 子项）/`### spaLoaderTag`/`### teleportAttrs`/`### teleportId`/`### teleportTag` 八节全部记载：各自说明句、类型与默认值，含 `rootAttrs` 与 `rootId` 说明句重复这一处、`spaLoaderAttrs` 说明句里的 "SPA loading template" 措辞；相邻的 `## spaLoadingTemplate` 节）
- https://nuxt.com/docs/4.x/api/components/teleports（`to` 需要 CSS 选择器字符串或 DOM 节点、SSR 只支持 `#teleports`、其它目标需 `<ClientOnly>` 的 `::warning` 与示例）
