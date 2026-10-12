---
url: /posts/d5b109d6a1bb659af49ea8b25289f332/
title: titleTemplate 与动态标题：函数为什么不能写在 nuxt.config
date: 2027-03-23T11:00:00+08:00
updated: 2027-03-23T11:00:00+08:00
author: cmdragon

summary:
  摘要：`titleTemplate` 有两种形态——带 `%s` 占位的字符串，或一个函数；官方的限制句很干脆：**要用函数（full control）就不能写在 `nuxt.config` 里，推荐写进 `app.vue`，这样会作用于全站所有页面**。原因是"标题在 `nuxt.config` 里没法是动态的"。模板函数拿到的入参是页面自己给的那段标题，传 `null` 时落到假分支的 `'Site Title'`。除 `%s` 之外还有 `templateParams` 提供额外占位符。另一条硬限制在建构期：`definePageMeta` 里的 `title` 是**通过宏在构建期提取的，因此不能被动态设置**，官方给的读法是布局里用 `useRoute` 取 `route.meta.title`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 页面标题
  - titleTemplate
  - templateParams
  - definePageMeta
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

把 `titleTemplate` 写成函数放进 `nuxt.config.ts`，编辑器不报错，跑起来标题却一直是兜底值。

这不是 bug。官方在 SEO 那一页对这一项有明确的位置限制，而限制的成因与 `definePageMeta` 那条构建期规则是同一类事情。

## 两种形态与一条位置限制

官方的定义段：

> You can use the `titleTemplate` option to provide a dynamic template for customizing the title of your site. For example, you could add the name of your site to the title of every page.
>
> The `titleTemplate` can either be a string, where `%s` is replaced with the title, or a function.

形态两种：**字符串里的 `%s` 会被标题替换**，或者**一个函数**。

紧接着是这一篇的正题：

> If you want to use a function (for full control), then this cannot be set in your `nuxt.config`. It is recommended instead to set it within your `app.vue` file where it will apply to all pages on your site:

三个动作写进同一句：函数形态**不能设在 `nuxt.config`**、**推荐放在 `app.vue`**、放在 `app.vue` 的好处是**作用于站点所有页面**。

官方给的示例：

```vue [app.vue]
<script setup lang="ts">
useHead({
  titleTemplate: (titleChunk) => {
    return titleChunk ? `${titleChunk} - Site Title` : 'Site Title'
  },
})
</script>
```

示例之后那句解释了两个分支各自什么时候被走到：

> Now, if you set the title to `My Page` with `useHead` on another page of your site, the title would appear as 'My Page - Site Title' in the browser tab. You could also pass `null` to default to 'Site Title'.

读法：**入参就是页面自己给的那段标题**（示例里叫 `titleChunk`，参数名随意），有值时拼成"页面 - 站点名"，**传 `null` 就走假分支**。`useHead` 参数表里 `titleTemplate` 的类型写成 `string | ((title?: string) => string)`，那个 `title?` 就是"参数可能为空"的来源。

## 为什么 nuxt.config 不行

Dynamic Title 那一节把原因单独写了一句：

> `nuxt.config` is also used as an alternative way of setting the page title. However, `nuxt.config` does not allow the page title to be dynamic. Therefore, it is recommended to use `titleTemplate` in the `app.vue` file to add a dynamic title, which is then applied to all routes of your Nuxt app.

关键在 **does not allow the page title to be dynamic**：这一层做不到动态，所以动态标题只能去 `app.vue`。这与 486 记的那条 `::important` 是同一原则的两次出现——`app.head` 那层不提供响应式数据，官方都建议改用 `useHead()`。

两种形态各有一份示例。字符串那份是最简写法：

```vue [app.vue]
<script setup lang="ts">
useHead({
  // as a string,
  // where `%s` is replaced with the title
  titleTemplate: '%s - Site Title',
})
</script>
```

函数那份的差别是**能拿到入参做判断**（官方说这份示例"allows greater flexibility in setting the page title dynamically"，并把参数名写成 `productCategory`，用来强调它只是页面传进来的那段值）。

## 除 `%s` 之外还能有别的占位符

`templateParams` 一节的原文：

> You can use `templateParams` to provide additional placeholders in your `titleTemplate` besides the default `%s`. This allows for more dynamic title generation.

示例把站点名与分隔符都提成参数：

```vue [app.vue]
<script setup lang="ts">
useHead({
  titleTemplate: (titleChunk) => {
    return titleChunk ? `${titleChunk} %separator %siteName` : '%siteName'
  },
  templateParams: {
    siteName: 'Site Title',
    separator: '-',
  },
})
</script>
```

两点要留意：**`%s` 是"默认占位符"**，`templateParams` 加的是它之外的；以及函数形态下这些 `%xxx` 出现在**你自己拼的字符串里**（示例的假分支返回字面量 `'%siteName'`）。**这些占位符在函数形态下由谁替换，官方没有明写**（文档未说明），要用时先跑一次看最终标题。

## definePageMeta 的 title 是构建期的

这一节和标题有关，但限制更硬。官方原话：

> For example, you can first set the current page title (this is extracted at build time via a macro, so it can't be set dynamically)

配套的两段示例是"页面写死、布局来读"：

```vue [pages/some-page.vue]
<script setup lang="ts">
definePageMeta({
  title: 'Some Page',
})
</script>
```

```vue [layouts/default.vue]
<script setup lang="ts">
const route = useRoute()

useHead({
  meta: [{ property: 'og:title', content: `App Name - ${route.meta.title}` }],
})
</script>
```

`definePageMeta` 的属性由**构建期宏提取**，所以 `title` 只能是静态字面量、不能运行时算出来。要读它就用 `route.meta.title`——官方这个示例把它填进了 `og:title`。

这条与 441 记过的静态提取清单、468 讲的 E1007（宏会被 transform 掉、绝不能动态调用）是同一套机制的三处表现。

## 和播报器的关系（一句话回指）

481 与 483 那条链的落点在这里：`<NuxtRouteAnnouncer>` 读的就是 Unhead 渲出的 `<title>`。**`titleTemplate` 拼出来的最终字符串就是会被播出去的那句话**——两条路由拼完仍同名，播报等于没装。

## 自查

- 函数形态不能放 `nuxt.config`，官方位置是 `app.vue`（作用于全站）；原因是标题在配置层做不到动态。
- 模板函数的入参是页面给的那段标题、可能为空；传 `null` 走假分支，所以官方两份示例都写了三元式。
- `templateParams` 提供 `%s` 之外的占位符；函数形态下这些 `%xxx` 由谁替换官方没写。
- `definePageMeta({ title })` 是构建期宏提取的静态值，不能动态设置；读它是 `route.meta.title`。
- 最终 `<title>` 就是播报内容，这一层写错，无障碍那几篇的努力会白废一半。

## 下一篇

下一篇 489 收这一页剩下的两节：滚动行为（默认三种：新路由滚到顶、返回时恢复位置、命中锚点滚动；改法是 `scrollBehaviorType` 或自己在 `router.options.ts` 里写 `scrollBehavior`，并要尊重 `prefers-reduced-motion`）与页末列出的三条外部资源。

## 本文依据

- SEO 页 Title Template 一节：定义两句（可字符串 `%s` 或函数、"this cannot be set in your `nuxt.config`"、推荐写在 `app.vue` 并作用于全站）、`useHead` 的函数示例、"My Page - Site Title" 与传 `null` 落兜底两句（`getting-started/seo-meta`）
- Template Parameters 一节：定义句与 `templateParams: { siteName, separator }` 示例
- Dynamic Title 一节：字符串与函数两份示例、"nuxt.config does not allow the page title to be dynamic. Therefore, it is recommended to use `titleTemplate` in the `app.vue` file"
- With `definePageMeta` 一节："this is extracted at build time via a macro, so it can't be set dynamically"、`definePageMeta({ title })` 与布局里 `route.meta.title` 填 `og:title` 的两段示例
- `useHead` 参数表：`title`（Sets the page title）与 `titleTemplate` 的类型 `string | ((title?: string) => string)`（`api/composables/use-head`）
- `app.head` 那条 `::important`（不能放响应式数据、建议 `useHead()`）与 `title: 'Nuxt'` 的 "default fallback title" 注释（`getting-started/seo-meta`）
- 宏被构建期 transform（`errors/e1007`）、`definePageMeta` 的静态提取键（`api/nuxt-config`）、播报器读 Unhead 标题（`guide/best-practices/accessibility`）
