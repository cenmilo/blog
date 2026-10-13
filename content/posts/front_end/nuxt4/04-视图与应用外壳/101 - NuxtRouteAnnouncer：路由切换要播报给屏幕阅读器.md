---
url: /posts/cb215d178b9400162afcb93ca61e24db/
title: 101 - NuxtRouteAnnouncer：路由切换要播报给屏幕阅读器
date: 2026-11-14T11:00:00+08:00
updated: 2026-11-14T11:00:00+08:00
author: cmdragon

summary:
  摘要：这一组件补的是客户端路由的一个空洞，官方的无障碍指南把原因写得很直白：**屏幕阅读器会自己播报整页加载，但它们没有办法知道发生了一次客户端导航**；一旦应用完成水合，路由就在客户端进行，document 永不再重新加载，浏览器**不再**替你播报新页面、也不再替你重置焦点。`<NuxtRouteAnnouncer>` 的做法是"渲染一个隐藏的 live region，并在每次导航之后把新的页面标题写进去"，组件页的描述则是"添加一个带页面标题的隐藏元素，向辅助技术播报路由变化"；它在 **Nuxt v3.12+** 可用。放置位置官方给的是 `app.vue` 或 `app/layouts/`，示例里它与页面出口**并列**而不是包在外面。它有一个默认插槽，可以从中拿到 `{ message }` 来输出自定义内容；两个 prop 分别是 `atomic`（控制屏幕阅读器是只播报变化还是播报全部内容：更新时要完整内容朗读就设 true，只播报变化就用 false，默认 `false`）与 `politeness`（设定播报的紧急程度：`off` 禁用播报、`polite` 等待静音、`assertive` 立即打断，默认 `polite`）。有一条限制比配置更重要：**announcer 读的是 Unhead 渲染出来的那个标题，所以它只在你的标题有价值时才有价值——如果两个路由共用同一个 `<title>`，用户在它们之间切换时什么都听不到**，而官方在页面标题一节把"给每个路由一个不同的标题"列为"单件最有价值的事"。要播报别的内容或改紧急程度就用 `useRouteAnnouncer`（官方示例里取 `const { set } = useRouteAnnouncer()` 再 `set(...)`），而**页内**变化（表单校验、toast、加载状态）官方要求改用 `<NuxtAnnouncer>` 配合 `useAnnouncer`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 无障碍
  - 屏幕阅读器
  - live-region
  - 路由

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


用屏幕阅读器点进一个 SPA，翻页时常常一片安静——页面换了，但没人为他说。这不是样式问题，是客户端路由天然丢掉的一个信号。`<NuxtRouteAnnouncer>` 就是补这个信号的组件。

## 为什么需要它

官方在无障碍指南开头把原因写得很具体：

| 官方原话 | 含义 |
| --- | --- |
| "屏幕阅读器会自己播报一次完整的页面加载，但**它们没有办法知道发生了一次客户端导航**。" | 问题定位 |
| "一旦你的应用完成了水合，路由就在客户端进行，于是 document 永远不会被重新加载，浏览器也不再替你播报新页面或重置焦点。" | 水合之后，这两个默认行为都消失了 |
| "`<NuxtRouteAnnouncer>` 通过在**每次导航之后**，把一个隐藏的 live region 渲染出来、并把新的页面标题写进它来解决这个问题。" | 它做的事就这两件 |
| 组件页描述："添加一个带页面标题的**隐藏元素**，向辅助技术播报路由变化。" | 视觉上看不见，只给读屏软件用 |

**它是 v3.12+ 才有的组件**（官方以 important 标注）。

## 放在哪儿：与页面出口并列

官方的用法说明："在你的 `app.vue` 或 `app/layouts/` 里添加 `<NuxtRouteAnnouncer/>`，通过告知辅助技术页面标题的变化来增强无障碍性。这确保了导航变化会被播报给依赖屏幕阅读器的用户。"

官方示例的位置关系很明确——**它是和布局/页面出口并列的一个兄弟节点，不是包在外面的壳**：

```vue [app/app.vue]
<template>
  <NuxtRouteAnnouncer />
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

无障碍指南里给的最小写法同样是这样两兄弟：

```vue [app.vue]
<template>
  <NuxtRouteAnnouncer />
  <NuxtPage />
</template>
```

## 一条限制比任何配置都重要

官方紧接着写了这句：**"announcer 读取的是 Unhead 渲染出来的那个标题，所以它只和你的标题一样有用。如果两个路由共用同一个 `<title>`，用户在它们之间什么都听不到。"**

而"Page Titles"一节的结论是：**因为路由 announcer 跟随 document 标题，给每一个路由一个不同的标题是你能做的最有价值的一件事。**

把这两句合起来看，用这个组件的实际工作量排序是这样的：

| 顺序 | 要做的事 | 不做会怎样 |
| --- | --- | --- |
| 1 | 让每个路由的 `<title>` 不同 | 加了组件也听不出变化——播报内容相同 |
| 2 | 把组件放进 `app.vue` 或布局 | 客户端导航完全没有播报 |
| 3 | 调 `politeness` / `atomic` | 只是体验差异，不是有无问题 |

**第 1 步不是这个组件的事，但它决定了第 2 步有没有意义。**

## 默认插槽：换掉播报出来的内容

官方说明："你可以通过 route announcer 的默认插槽传入自定义 HTML 或组件。"

```vue
<template>
  <NuxtRouteAnnouncer>
    <template #default="{ message }">
      <p>{{ message }} was loaded.</p>
    </template>
  </NuxtRouteAnnouncer>
</template>
```

插槽参数只有 `message` 一个，它就是这次要播报的那条文本。

## 两个 prop：紧急程度与是否朗读全文

| prop | 官方说明 | 可选值 | 默认 |
| --- | --- | --- | --- |
| `atomic` | 控制屏幕阅读器是**只播报变化**还是**播报整个内容**。更新时需要完整内容朗读就设为 true，只播报变化就用 false | `true` / `false` | **`false`** |
| `politeness` | 设定屏幕阅读器播报的**紧急程度** | `off`（禁用播报）、`polite`（等待静音）、`assertive`（立即打断） | **`polite`** |

怎么选可以按"这次变化能不能等"来判：

| 场景 | `politeness` |
| --- | --- |
| 普通换页 | `polite`（默认即可） |
| 完全不需要播报 | `off` |
| 必须打断用户的操作（例如会话失效） | `assertive` |

`atomic` 的判断标准是"变了哪一句"和"整块内容"哪个更有用——默认 `false` 只播报变化，设成 `true` 则在更新时朗读完整内容。

## 想播报别的内容：`useRouteAnnouncer`

官方的说明是"当你需要播报别的东西，或者改变它被播报的紧急程度时，使用 `useRouteAnnouncer`"，示例是搜索页在结果变化时播报数量：

```vue [app/pages/search.vue]
<script setup lang="ts">
const { set } = useRouteAnnouncer()
const { data: results } = await useFetch('/api/search')

watch(results, (results) => {
  set(`${results?.length ?? 0} results found`)
})
</script>
```

组件页也有一条对应提示：**你可以用 `useRouteAnnouncer` 这个组合式函数钩入底层 announcer 实例，它允许你设置一条自定义播报消息。**

## 页内变化不是它管的

这是最容易用错的一处，两处文档都写了同一句：

- 组件页 callout："对于播报**页内**内容变化（表单校验、toast 通知、加载状态等），请改用 `<NuxtAnnouncer>` 组件配合 `useAnnouncer` 组合式函数。"
- 无障碍指南："对于那些不是导航的页内更新，例如表单校验或 toast，请改用 `<NuxtAnnouncer>` 配合 `useAnnouncer`。"

分界就是这三个字：**是不是导航**。

| 你要播报的 | 用哪个 |
| --- | --- |
| 换了一个路由 | `<NuxtRouteAnnouncer>`（自动读标题） |
| 换路由之外还要补一句自定义消息 | `useRouteAnnouncer` 的 `set` |
| 表单校验结果、toast、加载完成 | `<NuxtAnnouncer>` + `useAnnouncer` |

## 它可选，也可以自己写

官方在组件页挂了两句：

- **这个组件是可选的。**
- 要达到完全自定义，你可以基于它自己的源代码实现你自己的一个。（Source 指向 `packages/nuxt/src/app/components/nuxt-route-announcer.ts`）

另外官方在指南开头还提了一个开发期工具：`@nuxt/a11y` 会在你开发时把组件里的无障碍问题暴露出来，但它处于 alpha，API 预期会变动。

## 自查

1. 为什么水合之后浏览器不再自动播报新页面？官方说的两件消失的默认行为是什么？
2. `<NuxtRouteAnnouncer>` 具体做哪两件事？从哪个版本起可用？
3. 官方给的放置位置是哪里？它和 `<NuxtPage>` 是什么结构关系？
4. 加了组件却什么都听不到，官方点明的第一原因是什么？
5. `politeness` 的三个取值分别是什么意思？默认哪个？
6. 表单校验的提示该用哪个组件？为什么不是路由 announcer？

## 下一篇

102 讲 `<NuxtLoadingIndicator>`：它的六个 prop（含 `color` 可以设成 `false`、`estimatedProgress` 收 `duration` 与 `elapsed` 返回 0-100）、`error` 被官方列为状态而不是 prop，以及怎么自己触发 start/finish。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-route-announcer`
  - 原文（描述）："The \<NuxtRouteAnnouncer> component adds a hidden element with the page title to announce route changes to assistive technologies."
  - 原文（important）："This component is available in Nuxt v3.12+."
  - 原文（Usage）："Add `<NuxtRouteAnnouncer/>` in your `app.vue` or `app/layouts/` to enhance accessibility by informing assistive technologies about page title changes. This ensures that navigational changes are announced to users relying on screen readers."
  - 原文示例：`<NuxtRouteAnnouncer />` 与 `<NuxtLayout><NuxtPage /></NuxtLayout>` 并列
  - 原文（Slots）："You can pass custom HTML or components through the route announcer's default slot."；示例 `<template #default="{ message }"><p>{{ message }} was loaded.</p></template>`
  - 原文（Props）："`atomic`: Controls if screen readers only announce changes or the entire content. Set to true for full content readouts on updates, false for changes only. (default `false`)"；"`politeness`: Sets the urgency for screen reader announcements: `off` (disable the announcement), `polite` (waits for silence), or `assertive` (interrupts immediately). (default `polite`)"
  - 原文（callout）："This component is optional."；"To achieve full customization, you can implement your own one based on its source code."
  - 原文（callout）："You can hook into the underlying announcer instance using the `useRouteAnnouncer` composable, which allows you to set a custom announcement message."
  - 原文（callout）："For announcing in-page content changes (form validation, toast notifications, loading states, etc.), use the `<NuxtAnnouncer>` component with the `useAnnouncer` composable instead."
- `https://nuxt.com/docs/4.x/guide/best-practices/accessibility`
  - 原文："Once your app has hydrated, it routes on the client, so the document is never reloaded and the browser no longer announces a new page or resets focus for you."
  - 原文（Route Announcements）："Screen readers announce a full page load by themselves, but they have no way of knowing that a client-side navigation happened. `<NuxtRouteAnnouncer>` solves this by rendering a hidden live region and writing the new page title into it after every navigation"；示例 `<NuxtRouteAnnouncer />` 与 `<NuxtPage />` 并列（文件标注 `app.vue`）
  - 原文："The announcer reads the title that Unhead rendered, so it is only as useful as your titles are. If two routes share the same `<title>`, users hear nothing on the way between them."
  - 原文："When you need to announce something else, or to change how urgently it is announced, use `useRouteAnnouncer`"；示例 `const { set } = useRouteAnnouncer()`、`watch(results, ...)`、``set(`${results?.length ?? 0} results found`)``
  - 原文（Page Titles）："Because the route announcer follows the document title, giving every route a distinct title is the single most valuable thing you can do."
  - 原文："For in-page updates that are not navigations, such as form validation or toasts, use `<NuxtAnnouncer>` with `useAnnouncer` instead."
  - 原文（tip）："`@nuxt/a11y` surfaces accessibility problems in your components while you develop, alongside the practices below. It is in alpha, so expect its API to change."
  - 原文（首段）："Most of what makes an app accessible is not specific to Nuxt: color contrast, form semantics, and ARIA work the same here as in any Vue or plain HTML application"
