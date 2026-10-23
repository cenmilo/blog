---
url: /posts/d1c2cd0b5c4305a6686f9371bd10295d/
title: 147 - NuxtLink props 全表：三组属性、internal 与 external 的判定
date: 2026-11-30T09:00:00+08:00
updated: 2026-11-30T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`<NuxtLink>` 的官方定性只有一句，但信息量全在里面："**`<NuxtLink>` is a drop-in replacement for both Vue Router's `<RouterLink>` component and HTML's `<a>` tag. It intelligently determines whether the link is *internal* or *external* and renders it accordingly with available optimizations (prefetching, default attributes, etc.)**"。它的 props 官方分三组列：**RouterLink 组**（`to`、`custom`、`exactActiveClass`、`activeClass`、`replace`、`ariaCurrentValue`，且前提是"**When not using `external`**, `<NuxtLink>` supports all Vue Router's `RouterLink` props"）、**NuxtLink 组**（`href`、`noRel`、`external`、`prefetch`、`prefetchOn`、`noPrefetch`、`prefetchedClass`）、**Anchor 组**（`target`、`rel`）。三组里最容易写错的是 `href`：官方定义是"**An alias for `to`. If used with `to`, `href` will be ignored**"——别名方向是 `href` 让位于 `to`，不是反过来。`external` 的定义是"**Forces the link to be rendered as an `<a>` tag instead of a Vue Router `RouterLink`.**"，而它要解决的问题官方说得很具体："By default, `<NuxtLink>` uses Vue Router's client side navigation for relative route. When linking to static files in the `/public` directory or to another application hosted on the same domain, **it might result in unexpected 404 errors** because they are not part of the client routes."。预取那一组里，`prefetchOn` 有两个取值（`visibility` 默认 / `interaction`），官方注明"**This prop is only used when `prefetch` is enabled (default) and `noPrefetch` is not set.**"，自定义触发时机"**We now support custom prefetch triggers for `<NuxtLink>` after `v3.13.0`.**"。默认值的覆盖口在 `experimental.defaults.nuxtLink`，官方列了八个键（含 `componentName: 'NuxtLink'`、`externalRelAttribute: 'noopener noreferrer'`、`trailingSlash: undefined`），并挂了一条将来会变的通知："**These options will likely be moved elsewhere in the future, such as into `app.config` or into the `app/` directory.**"

categories:
  - 前端开发

tags:
  - nuxt4
  - NuxtLink
  - props
  - 预取
  - 外部链接

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`<NuxtLink>` 用得最多，也最容易只在"能点"这个层次上用。它一份文档里同时挂着三组 props、一套内外部判定、一处全局默认值覆盖口。126 已经讲过 `rel`/`noRel` 那部分的安全含义，这一篇把整张属性表摊平，顺便把"什么时候它根本不是 RouterLink"说清楚。

## 定性句：它同时替代两个东西

> `<NuxtLink>` is a drop-in replacement for both Vue Router's `<RouterLink>` component and HTML's `<a>` tag. **It intelligently determines whether the link is *internal* or *external* and renders it accordingly with available optimizations (prefetching, default attributes, etc.)**

三个词值得抠：

| 词 | 落点 |
| --- | --- |
| drop-in replacement for **both** | 它既吃 vue-router 的 `RouterLink` props，也吃 `<a>` 的属性 |
| **intelligently determines** | 内外部是它自己判的，不是你声明的（`external` 是强制覆盖这个判断） |
| optimizations: **prefetching, default attributes** | 自动优化就两类：预取 + 默认属性（`rel` 那一类，126） |

渲染结果官方配了对比注释：`<NuxtLink to="/about">About page</NuxtLink>` 渲染成 `<!-- (Vue Router & Smart Prefetching) --> <a href="/about">About page</a>`。注意官方那行渲染结果：**内部链接渲染出来也是 `<a href="/about">`**，客户端导航只是拦住了左键点击，地址本身是真实可复制、可另开的。

## 三组 props：15 个

### 第一组：RouterLink 组

官方给这一组的前提句是："**When not using `external`, `<NuxtLink>` supports all Vue Router's `RouterLink` props**"。

| prop | 官方定义 |
| --- | --- |
| `to` | "Any URL or a route location object from Vue Router" |
| `custom` | "Whether `<NuxtLink>` should wrap its content in an `<a>` element. It allows taking full control of how a link is rendered and how navigation works when it is clicked. Works the same as Vue Router's `custom` prop" |
| `exactActiveClass` | "A class to apply on exact active links. Works the same as Vue Router's `exactActiveClass` prop on internal links. **Defaults to Vue Router's default (`"router-link-exact-active"`)**" |
| `activeClass` | "A class to apply on active links. Works the same as Vue Router's `activeClass` prop on internal links. **Defaults to Vue Router's default (`"router-link-active"`)**" |
| `replace` | "Works the same as Vue Router's `replace` prop on internal links" |
| `ariaCurrentValue` | "An `aria-current` attribute value to apply on exact active links. Works the same as Vue Router's `ariaCurrentValue` prop on internal links" |

三处 "on internal links" 是官方自己标的范围：`activeClass`/`exactActiveClass`/`replace`/`ariaCurrentValue` 这些行为只在内部链接上有意义——外部链接根本没有"当前路由"可比。

`to` 传对象时有一条编码规则，官方单独用 tip 写了出来：

> When you pass an object into the `to` prop, `<NuxtLink>` will inherit Vue Router's handling of query parameters. **Keys and values will be automatically encoded, so you don't need to call `encodeURI` or `encodeURIComponent` manually.**

命名路由 + params 的官方例子（与 112 那条同源）：

```vue [app/pages/index.vue]
<template>
  <NuxtLink :to="{ name: 'posts-id', params: { id: 123 } }">
    Post 123
  </NuxtLink>
</template>
```
```html [(Renders as) index.html]
<a href="/posts/123">Post 123</a>
```

官方还补了一句排查入口："Check out the Pages panel in Nuxt DevTools to see the route name and the params it might take."

### 第二组：NuxtLink 组

| prop | 官方定义 |
| --- | --- |
| `href` | "**An alias for `to`. If used with `to`, `href` will be ignored**" |
| `noRel` | "If set to `true`, no `rel` attribute will be added to the external link" |
| `external` | "**Forces the link to be rendered as an `<a>` tag instead of a Vue Router `RouterLink`.**" |
| `prefetch` | "When enabled will prefetch middleware, layouts and payloads (when using payloadExtraction) of links in the viewport. Used by the experimental crossOriginPrefetch config." |
| `prefetchOn` | "Allows custom control of when to prefetch links. Possible options are `interaction` and `visibility` (default). You can also pass an object for full control, for example: `{ interaction: true, visibility: true }`. **This prop is only used when `prefetch` is enabled (default) and `noPrefetch` is not set.**" |
| `noPrefetch` | "Disables prefetching." |
| `prefetchedClass` | "A class to apply to links that have been prefetched." |

`href` 那行是最容易写反的：**`href` 是 `to` 的别名，两者同时给时 `href` 被忽略**。写组件封装时如果同时透传了两个属性，实际生效的只有 `to`。

`prefetch` 与 `noPrefetch` 是一组开关，`prefetchOn` 只在"`prefetch` 开着且没设 `noPrefetch`"时才被读——这一句是官方在 `prefetchOn` 定义末尾直接写的优先级说明。

### 第三组：Anchor 组

| prop | 官方定义 |
| --- | --- |
| `target` | "A `target` attribute value to apply on the link" |
| `rel` | "A `rel` attribute value to apply on the link. **Defaults to `"noopener noreferrer"` for external links.**" |

这一组旁边挂着官方那条 tip："Defaults can be overwritten, see [overwriting defaults] if you want to change them."（`rel`、`noRel` 的安全侧读法在 126；两处并看才有完整图。）

## 内外部判定：什么算"外部"

官方把"外部"分成两类。第一类是**看起来内部但其实不是**的：

> By default, `<NuxtLink>` uses Vue Router's client side navigation for relative route. When linking to **static files in the `/public` directory** or to **another application hosted on the same domain**, it might result in unexpected 404 errors because they are not part of the client routes. In such cases, you can use the `external` prop with `<NuxtLink>` to bypass Vue Router's internal routing mechanism.

> The `external` prop explicitly indicates that the link is external. `<NuxtLink>` will render the link as a standard HTML `<a>` tag. This ensures the link behaves correctly, **bypassing Vue Router's logic** and directly pointing to the resource.

官方给了两个例子，形态完全一样，只是地址不同：

```vue [app/pages/index.vue]
<template>
  <NuxtLink to="/example-report.pdf" external>Download Report</NuxtLink>
</template>
```

```vue [app/pages/index.vue]
<template>
  <NuxtLink to="/another-app" external>Go to Another App</NuxtLink>
</template>
```

注意这两个地址：`/example-report.pdf` 和 `/another-app` **都是相对路径**。也就是说"相对路径 = 内部链接"这个默认直觉在 Nuxt 里是不成立的——**路径存在不等于路由表里有**。官方收尾那句是："Using the `external` prop or relying on automatic handling ensures proper navigation, avoids unexpected routing issues, and improves compatibility with static resources or cross-application scenarios."

第二类是**绝对地址**，官方在外部路由示例里直接给出渲染结果：

```vue [app/app.vue]
<template>
  <NuxtLink to="https://nuxtjs.org">Nuxt website</NuxtLink>
  <!-- <a href="https://nuxtjs.org" rel="noopener noreferrer">...</a> -->
</template>
```

判定基准在 `rel` 那节写得更机械：默认加 `rel` 的是"带 `target` 属性的链接，或**绝对链接（以 `http://`、`https://` 或 `//` 开头）**"。**`//` 也算绝对链接**——协议相对写法会被当成外部处理。

## 预取那一套：两种触发时机与三处开关

官方对自动预取的描述是一段很长的原话，值得整段读：

> Nuxt automatically includes smart prefetching. That means it detects when a link is visible (by default), either in the viewport or when scrolling and prefetches the JavaScript for those pages so that they are ready when the user clicks the link. **Nuxt only loads the resources when the browser isn't busy and skips prefetching if your connection is offline or if you only have 2g connection.**

最后一句是这套机制"节制"在哪：浏览器忙就不加载，离线或 2g 直接跳过。

自定义触发时机（官方注明 "after `v3.13.0`"）：

```vue
<template>
  <NuxtLink prefetch-on="visibility">
    This will prefetch when it becomes visible (default)
  </NuxtLink>

  <NuxtLink prefetch-on="interaction">
    This will prefetch when hovered or when it gains focus
  </NuxtLink>
</template>
```

两值的官方定义：

- `visibility`："Prefetches when the link becomes visible in the viewport. Monitors the element's intersection with the viewport using the **Intersection Observer API**. Prefetching is triggered when the element is scrolled into view."
- `interaction`："Prefetches when the link is hovered or focused. This approach **listens for `pointerenter` and `focus` events**, proactively prefetching resources when the user indicates intent to interact."

对象写法与"两个都开"的代价，官方连着给：

```vue
<NuxtLink :prefetch-on="{ interaction: true }">…</NuxtLink>
```
> That you probably don't want both enabled!
```vue
<NuxtLink :prefetch-on="{ visibility: true, interaction: true }">…</NuxtLink>
```
> This configuration will observe when the element enters the viewport and also listen for `pointerenter` and `focus` events. **This may result in unnecessary resource usage or redundant prefetching, as both triggers can prefetch the same resource under different conditions.**

关预取有三种写法，官方并列展示：

```vue [app/pages/index.vue]
<NuxtLink to="/about" no-prefetch>
About page not pre-fetched
</NuxtLink>

<NuxtLink to="/about" :prefetch="false">
About page not pre-fetched
</NuxtLink>
```

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    defaults: {
      nuxtLink: {
        prefetch: false,
      },
    },
  },
})
```

跨源预取另开一个开关（experimental）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    crossOriginPrefetch: true,
  },
})
```

官方说明："To enable cross-origin prefetching, you can set the `crossOriginPrefetch` option in your `nuxt.config`. **This will enable cross-origin prefetching using the Speculation Rules API.**"

`custom` 与预取的冲突、以及插槽里怎么自己调 `prefetch()`，145 已经把原文与例子放全了；这一格只提醒一句：props 那节的 note 是官方硬规定——"With the `custom` prop, `prefetch`, `prefetchOn` and `prefetchedClass` **do not attach handlers or classes automatically**."

## 默认值覆盖：一处配置，八个键

官方给的完整覆盖块（"In Nuxt Config"）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    defaults: {
      nuxtLink: {
        // default values
        componentName: 'NuxtLink',
        externalRelAttribute: 'noopener noreferrer',
        activeClass: 'router-link-active',
        exactActiveClass: 'router-link-exact-active',
        prefetchedClass: undefined, // can be any valid string class name
        trailingSlash: undefined, // can be 'append' or 'remove'
        prefetch: true,
        prefetchOn: { visibility: true },
      },
    },
  },
})
```

同一节挂着一条 `::important`，别当作无关紧要的小字：

> These options will likely be moved elsewhere in the future, such as into `app.config` or into the `app/` directory.

**"active class" 这件事在三处可配**，很容易撞车，我把三处一次列清：

| 层 | 配法 | 官方措辞 |
| --- | --- | --- |
| vue-router 全局 | `nuxt.config` 的 `router.options.linkActiveClass` / `linkExactActiveClass` | 属于"只有 JSON 可序列化的七个选项"那一批（142） |
| NuxtLink 默认值 | `experimental.defaults.nuxtLink.activeClass` / `exactActiveClass` | "A default class to apply on active links. **Works the same as Vue Router's `linkActiveClass` option.** Defaults to Vue Router's default" |
| 单个链接 | `active-class` / `exact-active-class` prop | "Works the same as Vue Router's `activeClass` prop on internal links" |

官方的措辞把关系说了：第二层的 `defineNuxtLink` 版 `activeClass` "works the same as vue-router 的 `linkActiveClass` 选项"，也就是它影响的是**这个组件**当默认值用；第三层只影响**这一个链接**。至于三者同时给值时具体怎么合并，官方没写一张优先级表——真要覆盖，我的做法是只留一处配（一般留全局那层，个别链接用 prop），不去依赖未说明的合并顺序。

## 一张速查表

| 我想… | 用 |
| --- | --- |
| 链到本项目页面 | `to="/about"`（内部，渲染成 `<a href>`，自动预取） |
| 链到命名路由 | `:to="{ name: 'posts-id', params: { id: 123 } }"` |
| 链到 `/public` 里的文件 | `to="/x.pdf" external` |
| 链到同域的另一个应用 | `to="/another-app" external` |
| 新窗口打开 | `target="_blank"`（会自动补 `rel="noopener noreferrer"`） |
| 不要那个 `rel` | `no-rel`（且不能再写 `rel`） |
| 不预取这一条 | `no-prefetch` 或 `:prefetch="false"` |
| 不预取全部 | `experimental.defaults.nuxtLink.prefetch = false` |
| 悬停/聚焦时才预取 | `prefetch-on="interaction"` |
| 给预取过的链接加类 | `prefetchedClass`（`custom` 下不会自动加） |
| 完全自渲染 | `custom` + 插槽 `href`/`navigate`（预取自己调） |
| URL 末尾斜杠统一 | `trailingSlash: 'append' \| 'remove'`（其它值会被忽略） |

## 自查

1. 官方的定性句里，`<NuxtLink>` 同时替代哪两个东西？"智能判定"判的是什么？自动优化官方列了哪两类？
2. 内部链接渲染出来的标签是什么？这对右键"新标签页打开"意味着什么？
3. `href` 与 `to` 同时写，哪个生效？官方原话怎么说？
4. 哪些 props 官方明确标了 "on internal links"？为什么这几个对外部链接没意义？
5. 相对路径就一定是内部链接吗？官方给了哪两类反例？`external` 的定义是什么？
6. 默认会被加 `rel="noopener noreferrer"` 的两类链接分别是什么？`//` 开头的地址算哪一类？
7. `prefetchOn` 在什么前提下才被读取？两个触发值分别监听什么事件/API？官方对"两个都开"给的理由是什么？
8. `custom` 会影响哪三个 props 的自动行为？
9. active class 这一件事可以配几处？官方对 `experimental.defaults.nuxtLink` 那句"将来会搬走"的原话是什么？

## 下一篇

148 讲 `defineNuxtLink`：怎么用八个选项做一个自带默认值的链接组件，`componentName` 到底改的是什么名字，以及为什么模板里的名字来自文件名。
