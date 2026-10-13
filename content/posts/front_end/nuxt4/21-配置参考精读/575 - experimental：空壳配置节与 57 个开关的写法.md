---
url: /posts/283480248940e4bec2f0d0d61c9d2430/
title: 575 - experimental：空壳配置节与 57 个开关的写法
date: 2027-04-21T11:00:00+08:00
updated: 2027-04-21T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## experimental` 与 `## features`、`## future` 一样是空壳——**配置参考里这一节只有一条 read-more，没有子项、没有类型、没有默认值**。写法只有一种形状：`experimental: { 开关名: 值 }`。开关清单在实验特性那一节，按小节顺序数下来是 **57 个**，但其中**只有 17 条用固定句式写明了默认状态**（14 条 "This flag is enabled by default" ＋ 3 条 "disabled by default"），剩下要么正文另有说法（"enabled by default when `future.compatibilityVersion` is `5` or higher"），要么根本没写。那一节指路的两句里，"API documentation" 那个链接**指回它自己**，能查完整清单的是它给的源码路径。

categories:
  - 前端开发

tags:
  - nuxt4
  - 实验特性
  - 开关清单
  - 默认值
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

你想查 `experimental.typedPages` 的默认值，于是打开配置参考搜 `experimental`。找到的小节标题下面是空的，只有一个跳转链接。**这一节不列任何开关。**

## 配置页这一节只有一条链接

`## experimental` 的全部内容是一条 read-more："Learn more about Nuxt's experimental features."，目标是 `guide/going-further/experimental-features`。

**没有定性句、没有 `**Type**`、没有 `**Default**`、一个 `###` 子项都没有。** 它和 `## features`、`## future` 是同一档：三节都不记载内容，只负责把你送到讲解页。区别在锚点——`features` 和 `future` 两条指向特性页的**同一个锚点**，这一条指向实验特性页本身。

配置里怎么写，文档只给一种形状（实验特性页 57 个小节里每节都自带一个这样的块）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    typedPages: true,
  },
})
```

## 那一节怎么指路，以及指路句里的一处自指

实验特性那一节开头三句是这一节的使用说明：

> Nuxt includes experimental features that you can enable in your configuration file.

> Internally, Nuxt uses `@nuxt/schema` to define these experimental features. You can refer to the [API documentation] or the [source code] for more information.

这里有一处要提醒：**那句 "API documentation" 的链接地址就是这一节自己**（`guide/going-further/experimental-features`）。也就是说想在配置参考里查实验开关的条目化记载，是查不到的——**这一节把自己当成了 API 文档**。可走的路径是它同时给的第二个链接，即源码文件：

```
packages/schema/src/config/experimental.ts
```

再加上一条 `::note` 定的风险范围：**"Note that these features are experimental and could be removed or modified in the future."**（可能被移除或被改动，而不是"稳定后可能改名"。）

## 57 个开关名，按页面顺序

下面是这一节 57 个小节的**全部名字**，顺序与那一节的排版一致，可以当索引对着查：

`alwaysRunFetchOnKeyChange`、`appManifest`、`asyncContext`、`asyncEntry`、`externalVue`、`extractAsyncDataHandlers`、`emitRouteChunkError`、`enforceModuleCompatibility`、`restoreState`、`inlineRouteRules`、`renderJsonPayloads`、`noVueServer`、`parseErrorData`、`payloadExtraction`、`clientNodePlaceholder`、`serverPathFallback`、`early404`、`clientFallback`、`crossOriginPrefetch`、`viewTransition`、`writeEarlyHints`、`componentIslands`、`localLayerAliases`、`typedPages`、`watcher`、`sharedPrerenderData`、`clientNodeCompat`、`scanPageMeta`、`cookieStore`、`buildCache`、`checkOutdatedBuildInterval`、`extraPageMetaExtractionKeys`、`extractSerializablePageMeta`、`navigationRepaint`、`navigateToEarlyReturn`、`normalizeComponentNames`、`normalizePageNames`、`spaLoadingTemplateLocation`、`browserDevtoolsTiming`、`debugModuleMutation`、`lazyHydration`、`templateImportResolution`、`templateRouteInjection`、`decorators`、`defaults`、`purgeCachedData`、`prefetchPreloadTags`、`granularCachedData`、`stripNeverHydratedData`、`headNext`、`routeTypedFetch`、`strictRouteTypes`、`pendingWhenIdle`、`entryImportMap`、`typescriptPlugin`、`viteEnvironmentApi`、`ssrStreaming`。

数一下是 57 个。这一节之外还有没有第 58 个开关，得去看 `@nuxt/schema` 那份源码——**因为文档侧的清单只有这一节的小节标题**。

## 默认状态只有 17 条写成固定句式

这 57 个小节的写法并不统一。用固定句式写明默认状态的只有两类：

- **"This flag is enabled by default"** —— 出现 **14** 次。例如 `appManifest` 那条就是"This flag is enabled by default, but you can disable this feature:"，然后给你一份写 `false` 的示例。
- **"This flag is disabled by default"** —— 出现 **3** 次。例如 `alwaysRunFetchOnKeyChange` 那条："This flag is disabled by default, but you can enable this feature:"，示例写 `true`。

**14 ＋ 3 = 17，也就是说 57 条里有 40 条没有用这个句式。** 它们的默认状态分三种情形：

1. 正文用别的说法交代，例如 "Enabled by default when `future.compatibilityVersion` is `5` or higher."、"This flag is enabled when `future.compatibilityVersion` is set to `5` or higher, but you can also enable it explicitly."。
2. 只在描述里隐含，例如某条写"你可以关掉它"但没说默认是开。
3. **完全不写**——这种只能实测或去读源码。

所以有一条实操判据：**不要凭"这是实验特性所以默认关"来推。** 文档里既默认开的（`appManifest`），也默认关的，还有跟着 `future.compatibilityVersion` 走的，三种混在一页里。要确定某个开关现在的状态，读它那一节的正文；读不出来就去那份 `experimental.ts`。

## 有一面旗在这一节上没有小节

升级页讲 `future.compatibilityVersion: 5` 那张十九项清单里，有一项写的是：

> **Error response format**: `experimental.inlineErrorRendering` is enabled, so page errors return the error page and server route errors return JSON

这个开关名在整套 4.x 文档里**只出现在升级页这一句**：实验特性页的 57 个小节里没有 `inlineErrorRendering`，配置参考里也没有。同理可对照的是那 19 项里**确实有**的六个——`viteEnvironmentApi`、`extractSerializablePageMeta`、`payloadExtraction`、`navigateToEarlyReturn`、`typedPages`、`routeTypedFetch` 都能在这一节找到同名小节。

结论要按字面给：**有一个开关被官方当作既成事实引用，但没有文档条目。** 需要它时，你只能按那一句写进 `experimental`，然后构建验证；不能指望在这一节查到它的默认值或取值范围。

## 自查

- 配置页 `## experimental`：**只有一条 read-more**，无子项、无类型、无默认值——想在本节查开关是查不到的。
- 写法只有一种：`experimental: { 开关名: 值 }`；示例散落在这一节的 57 个小节里。
- 那一节给的"API documentation"链接**指回这一节**；另一条指路是源码 `packages/schema/src/config/experimental.ts`。
- 开关共 **57** 个；**17 条**用固定句式写了默认状态（14 开 / 3 关），其余按三种情形分别处理。
- `experimental.inlineErrorRendering` 只在升级那一节出现过一次，**这一节无小节、配置参考里也无条目**。
- `::note` 的风险措辞是 "could be removed or modified in the future"。

## 下一篇

下一篇 576 结束这三个空壳节，进入有内容的 `## typescript`：**十一个子项**、`hoist` 那份 **17 项**默认清单、`strict` 默认 `true` 而 `shim` 默认 `false`，还有一处类型行与实际取值不一致的记载（`typeCheck` 写的是 `boolean`，正文却让你设成 `build`）。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## experimental` 仅一条 read-more；相邻的 `## features`、`## future` 两条 read-more 同锚点）
- https://nuxt.com/docs/4.x/guide/going-further/experimental-features（开场两句定性与指路句、"API documentation" 链接自指、源码路径 `packages/schema/src/config/experimental.ts`、`::note` 风险句、57 个小节标题、`appManifest` 与 `alwaysRunFetchOnKeyChange` 的默认状态句式示例、"enabled when `future.compatibilityVersion` is set to `5` or higher" 等措辞）
- https://nuxt.com/docs/4.x/getting-started/upgrade（十九项清单中 `experimental.inlineErrorRendering` 那一条，及其余六个能在这一节找到同名小节的开关）
