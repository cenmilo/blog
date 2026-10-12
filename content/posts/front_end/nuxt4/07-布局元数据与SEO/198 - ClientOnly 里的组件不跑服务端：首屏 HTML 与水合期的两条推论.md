---
url: /posts/12cd3612eed9de00ae10ec858a31b823/
title: ClientOnly 里的组件不跑服务端：首屏 HTML 与水合期的两条推论
date: 2026-12-17T09:00:00+08:00
updated: 2026-12-17T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：我犯的错是把 `<ClientOnly>` 只当"挡报错的壳"，包进去的组件里还顺手写了 `useHead`。这一篇先把文档写明的两句摆出来：一是 `<ClientOnly>` 页的 note——"The content of the default slot will be tree-shaken out of the **server** build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)"；二是同页 Accessing HTML Elements 一节——"Components inside `<ClientOnly>` are rendered only after being mounted."。由这两句可以往下推两条，但我必须标清它们是**推论而不是文档原话**：推论一，包在里面的组件如果在自己的 `setup` 里调 `useHead`，那段代码在服务端根本没跑，所以这些标签不可能出现在服务端 HTML 里；推论二，它们只会在水合之后出现在客户端 DOM 里，于是"不跑 JS 的抓取者看到的 head"和"用户看到的 head"就不一致——这正好撞上 hydration 页列的后果之一："**SEO problems**: Search engines may index different content than what users actually see."。文档在别处给了对照坐标：流式渲染那一节明确写了哪些位置注册的 head 能进 shell——"`useHead`/`useSeoMeta` called synchronously in `<script setup>` (before any `await`), in plugins, or in middleware render into the shell `<head>` as usual."。所以真要服务端就有标签，得把它挪出 `<ClientOnly>`。这一篇给了一条 `curl` 判据来证实上面两条。

categories:
  - 前端开发

tags:
  - nuxt4
  - ClientOnly
  - 水合
  - useHead
  - SSR

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先声明一点：`<ClientOnly>` 这一页**通篇没提 head**，我也不能替它编。这一篇的结构是——先抄文档写明的，再写我从这两句往下推的，最后给一条能证伪的验证方法。

## 文档写明的只有这些

`<ClientOnly>` 页的正文很短，跟"什么时候在服务端跑"相关的原话就三处：

> The `<ClientOnly>` component is used for purposely rendering a component only on client side.

> The content of the default slot will be tree-shaken out of the server build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)

> Components inside `<ClientOnly>` are rendered only after being mounted. To access the rendered elements in the DOM, you can watch a template ref

再加上 Props 与 Slots 两组：

| 项目 | 文档原文 |
| --- | --- |
| `placeholderTag` / `fallbackTag` | "specify a tag to be rendered server-side." |
| `placeholder` / `fallback` | "specify a content to be rendered server-side." |
| `#fallback` 插槽 | "specify a content to be rendered on the server and displayed until `<ClientOnly>` is mounted in the browser." |

注意最后一句里的时间描述：fallback 显示的是**到 `<ClientOnly>` 在浏览器挂载为止**。所以从服务端 HTML 到客户端挂载之间，页面上是 fallback；挂载之后才换成默认插槽的内容。CSS 那句也是同一件事的另一面——默认插槽不进服务端构建，它用的样式也就可能不内联进首屏 HTML。

顺带把官方那个"取 DOM"的写法留在这儿，因为它其实就是上面那句话的直接产物：

```vue [app/pages/example.vue]
<script setup lang="ts">
const nuxtWelcomeRef = useTemplateRef('nuxtWelcomeRef')

// The watch will be triggered when the component is available
watch(nuxtWelcomeRef, () => {
  console.log('<NuxtWelcome /> mounted')
}, { once: true })
</script>

<template>
  <ClientOnly>
    <NuxtWelcome ref="nuxtWelcomeRef" />
  </ClientOnly>
</template>
```

要 `watch` 一个模板 ref 才拿得到元素，本身就在说"渲染发生在挂载之后"。

## 推论一：壳里的 useHead 不可能进服务端 HTML

**这条是推论，文档没写。** 依据是上面两句：默认插槽的内容被摇树摇出服务端构建、壳里的组件只在挂载后渲染。既然它的 `setup` 在服务端没跑，那 `setup` 里的 `useHead({...})` 也就没被调用过。

文档在别处给了一把对照尺子，正好说明"哪些位置注册的标签能进首屏"——流式渲染那一节的原话（193 也引过）：

> The renderer waits for the first render pass before flushing the shell, so `useHead`/`useSeoMeta` called synchronously in `<script setup>` (before any `await`), in plugins, or in middleware render into the shell `<head>` as usual.

它点名的是**同步的 `setup`、插件、中间件**。一个根本不在服务端渲染的组件显然不在这个名单里。所以如果 head 标签需要服务端就有，注册点应该挪到壳外面：`app.vue`（SEO 页的推荐就是 "We recommend using `useHead()` in `app.vue`."）、布局（185 已经证明布局可以用 `useRoute` + `useHead`）、页面自身 `setup` 里在 `<ClientOnly>` 之外的那段，或者中间件/插件。

## 推论二：不跑 JS 的抓取者看到的 head 会更少

同样**是推论**。链条是：标签只在客户端出现 ⇒ 不执行 JS 的一方拿不到它们 ⇒ 抓取者与用户看到的 head 不一样。 hydration 页把这类后果写得很直白（它讲的是内容层面，但句子本身够用）：

> **SEO problems**: Search engines may index different content than what users actually see.

而流式那一节还给了同一族里的另一个"看不到"名单，句式和 193 里一样：

> Anything else (a crawler your `botRegex` does not list, a link unfurler, `curl`) will not see those tags at all.

这句原本是讲"晚注册的标签变成客户端补丁"的情形，但它把 `curl` 明确列为**拿不到的一侧**，所以我拿它当验证工具：`curl` 看到的就是不跑 JS 的一方看到的。

## 一次能把两条都证伪的验证

我用的是最笨但最可信的两步：

1. `nuxt build` + `nuxt preview`（不要用 dev，dev 的服务端 HTML 和生产不完全是一回事，165 里 `<DevOnly>` 那条教训同理），然后 `curl` 目标路由，在返回的 HTML 里搜你要的那个标签（比如 `name="description"` 或某条 `property="og:..."`）。**这一步验证推论一**：如果标签只在包了 `<ClientOnly>` 的版本里消失、挪到壳外就出现，那结论成立。
2. 同一条 URL 在浏览器里打开，等页面水合完成后再在开发者工具里搜一次 DOM。**这一步验证推论二**：`curl` 结果与浏览器 DOM 结果不一致，就是那句 "different content than what users actually see" 的具体形态。

顺手把对照组也做了：同一个页面上放一份 `#fallback`，看首屏 HTML 里出现的是 fallback 的文本还是壳内组件的文本——这能帮你确认自己看到的差异到底是"服务端没渲染"还是"水合还没跑完"。

## 自查

1. `<ClientOnly>` 页里哪两句支撑了"壳里的组件不在服务端渲染"？（答："tree-shaken out of the server build" 与 "rendered only after being mounted"。）
2. 文档点名能进 shell `<head>` 的三个注册位置是？（答：`<script setup>` 里在任何 `await` 之前的同步调用、插件、中间件。）
3. 我怎么区分"服务端没渲染"和"水合没完成"？（答：`curl` 看服务端 HTML，浏览器等水合后再看 DOM，两者对照；必要时看 `#fallback` 的文本落在哪一侧。）

## 下一篇

199 - 水合错配的代价里有一条是 SEO：五种成因与对应解法

## 本文依据

- `api/components/client-only`：定位句 "used for purposely rendering a component only on client side"；note "The content of the default slot will be tree-shaken out of the server build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)"；Props 两条与 Slots 一条（`#fallback` "...displayed until `<ClientOnly>` is mounted in the browser."）；Accessing HTML Elements 一段与 `useTemplateRef` + `watch(..., { once: true })` 示例。
- `guide/going-further/experimental-features`（streaming）："`useHead`/`useSeoMeta` called synchronously in `<script setup>` (before any `await`), in plugins, or in middleware render into the shell `<head>` as usual."；"Anything else (a crawler your `botRegex` does not list, a link unfurler, `curl`) will not see those tags at all."
- `guide/best-practices/hydration`："**SEO problems**: Search engines may index different content than what users actually see."；In Summary 第 2 条 "**Wrap client-only code**: Use `ClientOnly` component for browser-specific content"。
- `getting-started/seo-meta`：important "This method does not allow you to provide reactive data. We recommend using `useHead()` in `app.vue`."
