---
url: /posts/c745af455cdf1286b147024440524734/
title: 144 - useRouteAnnouncer 与 useAnnouncer：自动播报与手动播报
date: 2026-11-29T09:00:00+08:00
updated: 2026-11-29T09:00:00+08:00
author: cmdragon

summary:
  摘要：屏幕阅读器面对客户端导航是无感的，官方给的两句话定位分别是："**Screen readers announce a full page load by themselves, but they have no way of knowing that a client-side navigation happened.**" 与 "Use this for in-page updates like form validation, async operations, toast notifications, and live content changes."。两套 API 的版本门槛不同：`useRouteAnnouncer` 与 `<NuxtRouteAnnouncer>` 是 "**This composable is available in Nuxt v3.12+.**"，`useAnnouncer` 与 `<NuxtAnnouncer>` 是 "**This composable is available in Nuxt v4.4.2+.**"。`useRouteAnnouncer` 的机制官方写得很具体："A composable which observes the page title changes and updates the announcer message accordingly. Used by `<NuxtRouteAnnouncer>` and controllable. **It hooks into Unhead's `dom:rendered` hook to read the page's title and set it as the announcer message.**"——它读的是 Unhead 渲染出来的 `<title>`，于是有一句很实的后果："**If two routes share the same `<title>`, users hear nothing on the way between them.**"。`useAnnouncer` 则相反："Unlike `useRouteAnnouncer` which automatically announces route changes, `useAnnouncer` **gives you manual control over what and when to announce**."，并挂着一句必须满足的前提："**You need to add the `<NuxtAnnouncer>` component to your app for the announcements to be rendered in the DOM.**"。两者暴露的形状一模一样：属性 `message`、`politeness`，方法 `set(message, politeness = "polite")`、`polite(message)`、`assertive(message)`；`politeness` 三值是 `off`（disable the announcement）、`polite`（waits for silence）、`assertive`（interrupts immediately），默认 `polite`。官方那张对比表的四行是：Purpose（Announces route/page changes / Announces any dynamic content）、Trigger（Automatic on navigation / Manual via `polite()`/`assertive()`）、Message source（Page `<title>` / Developer-provided）、**atomic default（`false` / `true`）**——这一行是两个组件唯一的默认值分歧。

categories:
  - 前端开发

tags:
  - nuxt4
  - 无障碍
  - 播报
  - composables
  - 屏幕阅读器

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


101 那篇讲 `<NuxtRouteAnnouncer>` 这个组件时，我把它当成"装上就有播报"的东西。真正要控制播报的是它的两个 composable：一个跟着 `<title>` 走，一个完全由你说了算。它们的名字只差一个 `Route`，能力差别却很大。

## 先分家：两套不是新旧关系

官方在文档里互相指路，指得很清楚：

- `useRouteAnnouncer` 页的 callout：**"For announcing dynamic in-page content changes (form validation, toasts, loading states), use `useAnnouncer` instead."**
- `useAnnouncer` 页的 callout：**"For automatic announcements of route/page changes, use `useRouteAnnouncer` with the `<NuxtRouteAnnouncer>` component instead."**

| | 路由侧 | 页面内 |
| --- | --- | --- |
| composable | `useRouteAnnouncer` | `useAnnouncer` |
| 配套组件 | `<NuxtRouteAnnouncer>` | `<NuxtAnnouncer>` |
| 版本门槛 | v3.12+ | v4.4.2+ |
| 消息从哪来 | 页面 `<title>` | 你传进去的字符串 |
| 谁按下播报 | 导航完成后自动 | 你手动调 `polite()` / `assertive()` |

版本门槛这一行是官方的 `::important` 块原话：`useRouteAnnouncer` 页写 "This composable is available in Nuxt v3.12+."，`useAnnouncer` 页与 `<NuxtAnnouncer>` 页都写 "available in Nuxt v4.4.2+"。**如果你的项目版本低于 4.4.2，页面内那半套是不存在的**，只能先把内容变化塞进路由播报（下一节最后会讲这条路）。

## 播报为什么存在：客户端导航不会自动出声

官方在无障碍指南开篇讲清了动机，两句都值得记住：

> Once your app has hydrated, it routes on the client, so the document is never reloaded and the browser no longer announces a new page or resets focus for you.

> Screen readers announce a full page load by themselves, but they have no way of knowing that a client-side navigation happened. [`<NuxtRouteAnnouncer>`] solves this by **rendering a hidden live region and writing the new page title into it after every navigation**.

所以路由播报器的工作方式是"**隐藏 live region + 写页面标题**"，而不是弹个可见的提示。

## useRouteAnnouncer：它读的是 Unhead 渲染出来的标题

官方描述里藏着一个具体机制：

> A composable which observes the page title changes and updates the announcer message accordingly. Used by `<NuxtRouteAnnouncer>` and controllable. **It hooks into Unhead's `dom:rendered` hook to read the page's title and set it as the announcer message.**

三个信息点：

| 点 | 含义 |
| --- | --- |
| 它是"被 `<NuxtRouteAnnouncer>` 使用的" | 组件是壳，这个 composable 才是实例；你也可以在任意组件里拿到同一个实例 |
| 它监听 `dom:rendered` | 播报发生在标题**渲染之后**，所以标题是 `useHead`/`titleTemplate` 算出来的最终值 |
| 消息 = 页面标题 | 你没法指望它"知道"页面内容变了；它只知道标题变了没有 |

由此推出官方那句很实在的告诫：

> The announcer reads the title that Unhead rendered, so it is only as useful as your titles are. **If two routes share the same `<title>`, users hear nothing on the way between them.**

**这一句是整节里最能落地的一条**：101 讲过标题相同就静默。官方给的对治办法在"Page Titles"一节，直接放全局模板：

```vue [app.vue]
<script setup lang="ts">
useHead({
  titleTemplate: title => title ? `${title} - Nuxt` : 'Nuxt',
})
</script>
```

配合官方那句评价："Because the route announcer follows the document title, **giving every route a distinct title is the single most valuable thing you can do.**"

形状（参数、属性、方法）：

```vue [app/pages/index.vue]
<script setup lang="ts">
const { message, politeness, set, polite, assertive } = useRouteAnnouncer({
  politeness: 'assertive',
})
</script>
```

| 成员 | 官方定义 |
| --- | --- |
| 参数 `politeness` | "Sets the urgency for screen reader announcements: `off` (disable the announcement), `polite` (waits for silence), or `assertive` (interrupts immediately). (default `polite`)." |
| 属性 `message` | `Ref<string>` — The message to announce |
| 属性 `politeness` | `Ref<string>` — Screen reader announcement urgency level `off`, `polite`, or `assertive` |
| 方法 `set(message, politeness = "polite")` | Sets the message to announce with its urgency level. |
| 方法 `polite(message)` | Sets the message with `politeness = "polite"` |
| 方法 `assertive(message)` | Sets the message with `politeness = "assertive"` |

## 用 set 接管：把"结果数"播报出去

官方在无障碍指南里给 `set` 的例子不是导航，而是搜索结果变化：

```vue [app/pages/search.vue]
<script setup lang="ts">
const { set } = useRouteAnnouncer()
const { data: results } = await useFetch('/api/search')

watch(results, (results) => {
  set(`${results?.length ?? 0} results found`)
})
</script>
```

这一例给出的定位是：**`set` 让你盖过"标题"这个默认消息源**，改播你自己的内容。它对应官方那句话："When you need to announce something else, **or to change how urgently it is announced**, use `useRouteAnnouncer`."

于是老项目（低于 4.4.2、没有 `useAnnouncer`）里的替代做法就是这一条：借用路由播报器的手动入口。代价是它复用的是同一个 live region，标题播报与手动消息会互相顶掉。

## useAnnouncer：手动，且必须配一个组件

官方描述第一句就是对比：

> A composable for announcing dynamic content changes to screen readers. **Unlike `useRouteAnnouncer` which automatically announces route changes, `useAnnouncer` gives you manual control over what and when to announce.**

> Use this for in-page updates like form validation, async operations, toast notifications, and live content changes.

前提是那个 callout，别漏：

> **You need to add the `<NuxtAnnouncer>` component to your app for the announcements to be rendered in the DOM.**

也就是说 `useAnnouncer()` 只写状态，DOM 里那个隐藏 live region 由 `<NuxtAnnouncer/>` 提供。官方给的 `app.vue` 形态是把两个播报器都放上：

```vue [app/app.vue]
<template>
  <NuxtAnnouncer />
  <NuxtRouteAnnouncer />
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

形状与 `useRouteAnnouncer` 完全同构（参数 `politeness`、属性 `message`/`politeness`、方法 `set`/`polite`/`assertive`），只有一处类型写法不同：`useAnnouncer` 的 `politeness` 属性是 `Ref<'polite' | 'assertive' | 'off'>`，而 `useRouteAnnouncer` 那一节写的是 `Ref<string>`。另外 `useAnnouncer` 页给两个便捷方法补了使用判断：

- `polite(message)`：**"Use for non-urgent updates that can wait for the screen reader to finish its current task."**
- `assertive(message)`：**"Use for urgent updates that should interrupt the screen reader immediately."**

官方三个 Use Cases 都是"在 watch/提交里调一行"的形态：

```vue [app/pages/contact.vue]
<script setup lang="ts">
const { polite, assertive } = useAnnouncer()

async function submitForm () {
  try {
    await $fetch('/api/contact', { method: 'POST', body: formData })
    polite('Message sent successfully')
  } catch (error) {
    assertive('Error: Failed to send message')
  }
}
</script>
```

```vue [app/components/LoginForm.vue]
<script setup lang="ts">
const { assertive } = useAnnouncer()

function validateForm () {
  const errors = []
  if (!email.value) { errors.push('Email is required') }
  if (!password.value) { errors.push('Password is required') }

  if (errors.length) {
    assertive(`Form has ${errors.length} errors: ${errors.join(', ')}`)
    return false
  }
  return true
}
</script>
```

```vue [app/pages/dashboard.vue]
<script setup lang="ts">
const { polite } = useAnnouncer()

const { data, status } = await useFetch('/api/data')

watch(status, (newStatus) => {
  if (newStatus === 'pending') {
    polite('Loading data...')
  } else if (newStatus === 'success') {
    polite('Data loaded successfully')
  }
})
</script>
```

（第四个场景"Search Results"在 `useAnnouncer` 页也存在：`watch(results, newResults => polite(\`Found ${newResults.length} results\`))`。注意它和上面 `useRouteAnnouncer` 的例子是**同一件事的两种做法**——同一份结果数，可以选择播在路由播报器上，也可以播在页面内播报器上。别两边都播，那会读两遍。）

## 唯一的默认值分歧：atomic

官方那张对比表（`<NuxtAnnouncer>` 页的 "Differences from `<NuxtRouteAnnouncer>`"）：

| Aspect | `<NuxtRouteAnnouncer>` | `<NuxtAnnouncer>` |
| --- | --- | --- |
| **Purpose** | Announces route/page changes | Announces any dynamic content |
| **Trigger** | Automatic on navigation | Manual via `polite()`/`assertive()` |
| **Message source** | Page `<title>` | Developer-provided |
| **atomic default** | `false` | `true` |

`atomic` 的语义两节写得一致："Controls if screen readers announce only changes or the entire content. Set to true for full content readouts on updates, false for changes only."

- `false`：只播**变化的部分**。
- `true`：每次更新都把**整段内容**读一遍。

**默认值一个 `false` 一个 `true`，官方没有给理由**。但它对使用的影响是直接的：`<NuxtAnnouncer>` 默认整段重读，所以连续 `polite()` 多条消息时，前一条可能被打断重念；如果你在同一个 live region 里高频推消息（比如搜索打字），把 `:atomic="false"` 显式关掉更稳。这句话是我按默认值和语义推的使用建议，不是文档结论——真要上生产，用屏幕阅读器实测一遍再定。

## `set` 的第二个参数与实例默认值：一处需要你自己确认的细节

两个 composable 都提供 `politeness` 参数（官方称其为 "default urgency" / "the urgency"），而 `set` 的签名是 `set(message, politeness = "polite")`。文档只给了签名与默认值，**没有说明当你用 `useAnnouncer({ politeness: 'assertive' })` 改了默认、又只调用 `set(msg)` 时，实际生效的是哪一个**。这不是我该替你猜的：如果你依赖这个组合，实测一次——把 `:politeness` 显式传给 `set` 是最省心的写法，它绕开了这个歧义。

## 想换掉组件：两节都说是可选的

`<NuxtRouteAnnouncer>` 与 `<NuxtAnnouncer>` 页面都挂了同样的两条 callout：

> This component is optional. To achieve full customization, you can implement your own one based on its source code.

> You can hook into the underlying announcer instance using the [corresponding] composable, which allows you to set custom announcement message(s).

同时两节都提供默认插槽来自渲染内容：

```vue
<template>
  <NuxtAnnouncer>
    <template #default="{ message }">
      <p>{{ message }}</p>
    </template>
  </NuxtAnnouncer>
</template>
```

（`<NuxtRouteAnnouncer>` 的插槽例子写成 `<p>{{ message }} was loaded.</p>`，说明插槽里可以往消息上再加词。）

## 落到代码上的选择

| 场景 | 用 |
| --- | --- |
| 换页要出声 | `<NuxtRouteAnnouncer/>` + 保证每个路由标题唯一 |
| 表单校验错误、提交成功/失败 | `useAnnouncer()` 的 `assertive` / `polite` |
| loading 状态、数据加载完成 | `useAnnouncer()` 的 `polite`，配合 `watch(status, ...)` |
| 结果条数这类数量变化 | 二选一：`useAnnouncer().polite` 或 `useRouteAnnouncer().set`，**只播一处** |
| 项目版本 < 4.4.2，没有 `useAnnouncer` | 借 `useRouteAnnouncer().set` |
| 想完全自己控制 live region 的 DOM | 用默认插槽，或照官方说法基于源码自研一个 |

## 自查

1. 官方为什么强调"客户端导航屏幕阅读器本身无从知晓"？路由播报器的实现方式是哪两件事？
2. `useRouteAnnouncer` 读的消息来源是什么？它挂在 Unhead 的哪个钩子上？由此推出哪条关于 `<title>` 的硬后果？
3. `useRouteAnnouncer` 与 `useAnnouncer` 的版本门槛分别是什么？
4. `useAnnouncer` 能工作的前提是什么？那个组件提供的到底是什么？
5. 两者的 `politeness` 三个取值各自语义是什么？`polite()` 与 `assertive()` 官方给的用法判断是什么？
6. 官方对比表里两个组件唯一的默认值分歧是哪一项？两边默认分别是什么？
7. 低于 v4.4.2 的项目想做页面内播报，官方文档里哪条路可以借？代价是什么？
8. `set(message, politeness = "polite")` 与实例默认 `politeness` 的关系，文档说清了吗？你会怎么绕过这个歧义？

## 下一篇

145 讲 `preloadRouteComponents`：怎么手动预取下一条路由的组件，以及官方示例里为什么这句调用不做 `await`。

## 本文依据

- `https://nuxt.com/docs/4.x/api/composables/use-route-announcer`
  - 原文（页标题与描述）："This composable observes the page title changes and updates the announcer message accordingly."；"A composable which observes the page title changes and updates the announcer message accordingly. Used by `<NuxtRouteAnnouncer>` and controllable. It hooks into Unhead's `dom:rendered` hook to read the page's title and set it as the announcer message."
  - 原文（版本）："This composable is available in Nuxt v3.12+."
  - 原文（Parameters）："`politeness`: Sets the urgency for screen reader announcements: `off` (disable the announcement), `polite` (waits for silence), or `assertive` (interrupts immediately). (default `polite`)."
  - 原文（Properties）：`message` — "**type**: `Ref<string>`" / "The message to announce"；`politeness` — "**type**: `Ref<string>`" / "Screen reader announcement urgency level `off`, `polite`, or `assertive`"
  - 原文（Methods）："`set(message, politeness = "polite")` Sets the message to announce with its urgency level."；"`polite(message)` Sets the message with `politeness = "polite"`"；"`assertive(message)` Sets the message with `politeness = "assertive"`"
  - 原文（Example）：`const { message, politeness, set, polite, assertive } = useRouteAnnouncer({ politeness: 'assertive' })`（`app/pages/index.vue`）
  - 原文（callout）："For announcing dynamic in-page content changes (form validation, toasts, loading states), use `useAnnouncer` instead."
- `https://nuxt.com/docs/4.x/api/composables/use-announcer`
  - 原文（描述）："A composable for announcing dynamic content changes to screen readers. Unlike `useRouteAnnouncer` which automatically announces route changes, `useAnnouncer` gives you manual control over what and when to announce."；"Use this for in-page updates like form validation, async operations, toast notifications, and live content changes."
  - 原文（版本）："This composable is available in Nuxt v4.4.2+."
  - 原文（Parameters）："`politeness`: Sets the default urgency for screen reader announcements: `off` (disable the announcement), `polite` (waits for silence), or `assertive` (interrupts immediately). (default `polite`)"
  - 原文（Properties）：`message` — `Ref<string>` / "The current message to announce"；`politeness` — `Ref<'polite' | 'assertive' | 'off'>` / "Screen reader announcement urgency level"
  - 原文（Methods）："`set(message, politeness = "polite")`"；"`polite(message)` Sets the message with `politeness = "polite"`. Use for non-urgent updates that can wait for the screen reader to finish its current task."；"`assertive(message)` Sets the message with `politeness = "assertive"`. Use for urgent updates that should interrupt the screen reader immediately."
  - 原文（Example 与 Use Cases）：`app/pages/contact.vue` 的 `submitForm` 里 `polite('Message sent successfully')` / `assertive('Error: Failed to send message')`；`app/components/LoginForm.vue` 的 `assertive(\`Form has ${errors.length} errors: ${errors.join(', ')}\`)`；`app/pages/dashboard.vue` 的 `watch(status, ...)` → `polite('Loading data...')` / `polite('Data loaded successfully')`；`app/components/Search.vue` 的 `watch(results, (newResults) => { polite(\`Found ${newResults.length} results\`) })`
  - 原文（两条 callout）："You need to add the `<NuxtAnnouncer>` component to your app for the announcements to be rendered in the DOM."；"For automatic announcements of route/page changes, use `useRouteAnnouncer` with the `<NuxtRouteAnnouncer>` component instead."
- `https://nuxt.com/docs/4.x/api/components/nuxt-announcer`
  - 原文（描述与版本）："The `<NuxtAnnouncer>` component adds a hidden element to announce dynamic content changes to assistive technologies."；"This component is available in Nuxt v4.4.2+."
  - 原文（Usage）："Add `<NuxtAnnouncer/>` in your `app.vue` or `app/layouts/` to enable announcing dynamic content changes to screen readers. This is useful for form validation, toast notifications, loading states, and other in-page updates."；`app/app.vue` 示例把 `<NuxtAnnouncer />` 与 `<NuxtRouteAnnouncer />` 一起放在 `<NuxtLayout><NuxtPage /></NuxtLayout>` 之前
  - 原文（Slots）："You can pass custom HTML or components through the announcer's default slot." + `<template #default="{ message }"><p>{{ message }}</p></template>`
  - 原文（Props）："`atomic`: Controls if screen readers announce only changes or the entire content. Set to true for full content readouts on updates, false for changes only. (default `true`)"；"`politeness`: Sets the default urgency for screen reader announcements: `off` (disable the announcement), `polite` (waits for silence), or `assertive` (interrupts immediately). (default `polite`)"
  - 原文（Differences from `<NuxtRouteAnnouncer>` 表）：Purpose — "Announces route/page changes" / "Announces any dynamic content"；Trigger — "Automatic on navigation" / "Manual via `polite()`/`assertive()`"；Message source — "Page `<title>`" / "Developer-provided"；atomic default — "`false`" / "`true`"
  - 原文（两条 callout）："This component is optional. To achieve full customization, you can implement your own one based on its source code."；"You can hook into the underlying announcer instance using the `useAnnouncer` composable, which allows you to set custom announcement messages."
- `https://nuxt.com/docs/4.x/api/components/nuxt-route-announcer`
  - 原文（描述与版本）："The `<NuxtRouteAnnouncer>` component adds a hidden element with the page title to announce route changes to assistive technologies."；"This component is available in Nuxt v3.12+."
  - 原文（Usage）："Add `<NuxtRouteAnnouncer/>` in your `app.vue` or `app/layouts/` to enhance accessibility by informing assistive technologies about page title changes. This ensures that navigational changes are announced to users relying on screen readers."
  - 原文（Props）："`atomic`: ... (default `false`)"；"`politeness`: Sets the urgency for screen reader announcements: ... (default `polite`)"
  - 原文（插槽示例）：`<template #default="{ message }"><p>{{ message }} was loaded.</p></template>`
  - 原文（callout）："For announcing in-page content changes (form validation, toast notifications, loading states, etc.), use the `<NuxtAnnouncer>` component with the `useAnnouncer` composable instead."
- `https://nuxt.com/docs/4.x/guide/best-practices/accessibility`
  - 原文（开篇动机）："Once your app has hydrated, it routes on the client, so the document is never reloaded and the browser no longer announces a new page or resets focus for you."
  - 原文（Route Announcements）："Screen readers announce a full page load by themselves, but they have no way of knowing that a client-side navigation happened. `<NuxtRouteAnnouncer>` solves this by rendering a hidden live region and writing the new page title into it after every navigation."；"The announcer reads the title that Unhead rendered, so it is only as useful as your titles are. If two routes share the same `<title>`, users hear nothing on the way between them."；"When you need to announce something else, or to change how urgently it is announced, use `useRouteAnnouncer`."；示例 `const { set } = useRouteAnnouncer()` + `watch(results, (results) => { set(\`${results?.length ?? 0} results found\`) })`；"For in-page updates that are not navigations, such as form validation or toasts, use `<NuxtAnnouncer>` with `useAnnouncer` instead."
  - 原文（Page Titles）："Because the route announcer follows the document title, giving every route a distinct title is the single most valuable thing you can do. Set a global template in `app.vue` and let each page fill in its own part:" + `useHead({ titleTemplate: title => title ? \`${title} - Nuxt\` : 'Nuxt' })`
