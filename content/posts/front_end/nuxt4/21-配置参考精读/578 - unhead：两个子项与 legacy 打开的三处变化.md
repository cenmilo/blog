---
url: /posts/7754e836780406334300805aff392da9/
title: 578 - unhead：两个子项与 legacy 打开的三处变化
date: 2027-04-22T11:00:00+08:00
updated: 2027-04-22T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：页面元信息（`<title>`、`<meta>`、`<link>` 这些 `<head>` 里的标签）由 Unhead 这一层生成，而它在 `nuxt.config` 里的配置面**只有两个子项**。`## unhead` 的定性句是 "An object that allows us to configure the `unhead` nuxt module."；`legacy` 默认 `false`，打开它带来的三处变化被官方写在同一句里——**关闭 Capo.js 排序、加 `DeprecationsPlugin`（支持 `hid`/`vmid`/`children`/`body`）、加 `PromisesPlugin`（支持 promise 输入）**；`renderSSRHeadOptions` 默认 `{"omitLineBreaks": false}`，是传给 `renderSSRHead` 的透传对象。这三处变化正好对上升级页 "Unhead v2" 那节列的三条破坏性变更。

categories:
  - 前端开发

tags:
  - nuxt4
  - head管理
  - unhead
  - 兼容模式
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

从旧项目迁过来的 `metaInfo` 写法里，那种给标签加 `hid` 做去重、把 `children` 当内容塞进去的写法，在 4.x 里默认不认了。要让它们重新生效，入口是 `unhead.legacy`；而这个键在配置参考里只占三行。

## 这一节的定性句与两个子项

`## unhead` 的定性句："An object that allows us to configure the `unhead` nuxt module."

unhead 是什么，配置页没说；入门讲元信息那一节给了定性——**"Nuxt head tag management is powered by Unhead"**，即 `<head>` 里那些标签由这一层管理。`useHead` 那一节补了同一句的另一半：它是"以编程式且响应式的方式管理 head 标签"的引擎。

**配置里能动的只有两个子项**：`legacy` 与 `renderSSRHeadOptions`。除此之外这一节什么都没有——没有开关去改标签排序策略、也没有键能配 Unhead 的其他行为。

## legacy：三处变化写在同一句里

原文把三处变化连排写在同一句中（官方格式就是行内短横，不是一个真列表）：

> Enable the legacy compatibility mode for `unhead` module. This applies the following changes: - Disables Capo.js sorting - Adds the `DeprecationsPlugin`: supports `hid`, `vmid`, `children`, `body` - Adds the `PromisesPlugin`: supports promises as input

- **Type**: `boolean`
- **Default:** `false`

拆开就是三条：

| 变化 | 官方写法 | 影响 |
| --- | --- | --- |
| 排序 | "Disables Capo.js sorting" | 关掉按 Capo.js 优先级排序标签这件事 |
| 已废弃的 props | 加 `DeprecationsPlugin`，支持 `hid`、`vmid`、`children`、`body` | 这四个 props 重新可用 |
| 输入形态 | 加 `PromisesPlugin`，支持 promises as input | 可以传 Promise 当输入 |

示例：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  unhead: {
    legacy: true,
  },
})
```

并给了一条 See 指向 Unhead 的迁移文档（`unhead.unjs.io` 上的 get-started/migration）。

**这三条与升级页 "Unhead v2" 那节的三条是一一对着的。** 升级页说的是 Unhead 升到 v2 之后的破坏性变更：

- Removed props: `vmid`, `hid`, `children`, `body`.
- Promise input no longer supported.
- Tags are now sorted using Capo.js by default.

**换句话说：`unhead.legacy: true` 不是"打开旧功能"，而是把 v2 这三条变更按原样退回去。** 升级页给这一条的 **Impact Level 是 Minimal**，也就是默认不开也应能过去；真要开，你应该能指出是这三条里的哪一条在挡路——**是标签顺序变了、还是某个 `hid` 没起作用、还是传了 Promise 进去。**

## renderSSRHeadOptions：一个透传对象，文档只写了一个键

原文两句：

> An object that will be passed to `renderSSRHead` to customize the output.

- **Type**: `object`
- **Default**

```json
{
  "omitLineBreaks": false
}
```

示例只演示这一个键：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  unhead: {
    renderSSRHeadOptions: {
      omitLineBreaks: true,
    },
  },
})
```

`renderSSRHead` 是服务端把 head 渲染成字符串的那一步；这一项就是给它的参数。**`omitLineBreaks` 设成 `true` 之后的实际差别，文档没有描述**——从键名只能读出"换行"相关，具体输出差别要自己比对渲染结果。同理，**这个对象除了 `omitLineBreaks` 还能传什么，本节没有列**（它是透传，所以可传项由 Unhead 一侧决定）。

## 还有一处会让 legacy 失效

`future.compatibilityVersion: 5` 的那份十九项清单里有一条同时管着这里：

> Server-only head composables: `useServerHead`, `useServerHeadSafe` and `useServerSeoMeta` are no longer auto-imported, **`unhead.legacy` is ignored**, Unhead's legacy plugins are no longer registered and Capo sorting is always on

所以三者是绑在一起的：**开了 Nuxt 5 兼容档，`unhead.legacy` 就不再生效**，legacy 插件不再注册、Capo 排序恒开。**在配置里同时写 `future.compatibilityVersion: 5` 与 `unhead.legacy: true`，文档给的读法是"后者被忽略"**——它没有把这条冲突写成警告块，只在那一条清单句里出现。

## 自查

- `## unhead` 只有两个子项，**没有别的键可以改 head 生成行为**。
- `legacy` 默认 `false`；打开后的三处变化＝Unhead v2 那三条变更的反向（Capo 排序、四个废弃 props、Promise 输入）。
- 升级页给 Unhead v2 的影响级别是 **Minimal**，所以先确认自己撞的是三条里的哪一条再开。
- `renderSSRHeadOptions` 是传给 `renderSSRHead` 的透传对象，默认只有 `omitLineBreaks: false`；**其余可传项与该项的实际输出差别均未记载**。
- 与 `future.compatibilityVersion: 5` 同时存在时，`unhead.legacy` **被忽略**。

## 下一篇

下一篇 579 看 `## router`：这一节标题下**没有正文**，两个子项是 `options`（含那条"只能用可 JSON 序列化的选项，要更多控制去写 `router.options.ts`"的 callout）和它下面的 `hashMode`、`scrollBehaviorType`——后者在文档里**把默认值写了两遍**，而且没有列出可选值。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## unhead` 定性句；`### legacy` 三处变化原句、Type/Default、See 与示例；`### renderSSRHeadOptions` 两句、默认 JSON 与示例）
- https://nuxt.com/docs/4.x/getting-started/upgrade（"Unhead v2" 一节：Impact Level Minimal 与三条破坏性变更；十九项清单中 "Server-only head composables" 那一条关于 `unhead.legacy` 被忽略的措辞）
- https://nuxt.com/docs/4.x/getting-started/seo-meta（"Nuxt head tag management is powered by Unhead" 定性句）
- https://nuxt.com/docs/4.x/api/composables/use-head（head 标签管理"powered by Unhead"、以编程式且响应式的方式管理）
