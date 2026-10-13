---
url: /posts/dd3e1cafe3bf954ae2d0f7331c51847b/
title: 181 - name 与关闭：布局的四种不套外壳写法与错误页那一层
date: 2026-12-11T10:00:00+08:00
updated: 2026-12-11T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：094 讲完 `<NuxtLayout>` 的 `name`、`fallback`、props 与 `layoutRef`，这一篇只处理一件被拆在四页里的事：**怎么让某个页面不套外壳，以及怎么确认它真的没套**。`name` 的官方定义里已经含了第一种关法："Specify a layout name to be rendered, can be **a string, reactive reference or a computed property**. It **must** match the name of the corresponding layout file in the `app/layouts/` directory, **or `false` to disable the layout**."（type `string | false`，default `default`）。**第二种写在页面元数据里**：layouts 目录页的 "Overriding a Layout on a Per-page Basis" 一节——"If you are using pages, you can **take full control by setting `layout: false`** and then using the `<NuxtLayout>` component within the page."，示例是 `definePageMeta({ layout: false })` 配页面内 `<NuxtLayout name="custom">`。**第三种在路由规则里**（v4.3）：`routeRules: { '/landing': { appLayout: false } }`，与它同段的是 `'/admin': { appLayout: 'admin' }` 与 `'/dashboard/**': { appLayout: 'dashboard' }`，官方给这条的动机句是"This is useful when you want to **manage layouts centrally in your configuration** rather than in each page file, or when you need to **apply layouts to routes that don't have corresponding page components** (such as catchall pages which might match many paths)."。**第四种是意外的**："If an **invalid** layout is passed to the `name` prop, **no layout will be rendered**."——没有 `fallback` 时它是静默的。读回结果靠 `useLayout`：官方定义它返回"**the layout name (a `string`), or `false` when the layout is disabled**"。最后别漏了布局的第二类宿主："Nuxt provides the `<NuxtLayout>` component to show layouts **on pages and error pages**."，错误页那份示例是 `<NuxtLayout name="error-layout"><NuxtPage /></NuxtLayout>`，而名字要 kebab 是因为"the layout name is normalized to kebab-case, so if your layout file is named `errorLayout.vue`, it will become `error-layout`"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 布局
  - NuxtLayout
  - routeRules
  - 错误页

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


落地页要去掉头部导航、错误页要另一套外壳、后台整段共用一个布局——这三件事都是"不要默认外壳"，但官方把它们分散在四个地方，各自的写法不一样。这一篇收拢成一种清单。

## `name` 能接受的三种形态，与它的关法

官方定义：

> `name`: Specify a layout name to be rendered, can be a string, reactive reference or a computed property. It **must** match the name of the corresponding layout file in the `app/layouts/` directory, or `false` to disable the layout.
>
> - **type**: `string | false`
> - **default**: `default`

三种形态都合法，所以响应式的壳是成立的：

```vue [app/pages/index.vue]
<script setup lang="ts">
// layouts/custom.vue
const layout = 'custom'
</script>

<template>
  <NuxtLayout :name="layout">
    <NuxtPage />
  </NuxtLayout>
</template>
```

layouts 目录页那段同一形状的例子还带了一句取值来源的提示：

```vue [app/app.vue]
<script setup lang="ts">
// You might choose this based on an API call or logged-in status
const layout = 'custom'
</script>
```

**"logged-in status" 这句解释了为什么 `name` 要支持响应式引用**：布局可能取决于异步拿到的登录态，而不是写死在页面里。

`false` 是第一种关法（组件侧），但它有个限制：`<NuxtLayout>` 在 `app.vue` 里通常只有一份，把 `name` 设成 `false` 会影响所有页面。所以真正按页关闭要用第二、第三种。

## 第二种：页面里 `layout: false`，自己套壳

官方标题就是"Overriding a Layout on a Per-page Basis"，句子很短但给了完整控制权：

> If you are using pages, you can take full control by **setting `layout: false`** and then **using the `<NuxtLayout>` component within the page**.

官方示例是两段一起看的：

```vue [app/pages/index.vue]
<script setup lang="ts">
definePageMeta({
  layout: false,
})
</script>

<template>
  <div>
    <NuxtLayout name="custom">
      <template #header>
        Some header template content.
      </template>

      The rest of the page
    </NuxtLayout>
  </div>
</template>
```

```vue [app/layouts/custom.vue]
<template>
  <div>
    <header>
      <slot name="header">
        Default header content
      </slot>
    </header>
    <main>
      <slot />
    </main>
  </div>
</template>
```

三个可复用的点：

| 点 | 说明 |
| --- | --- |
| `layout: false` 之后页面可以**自己套布局** | 这一格是"关"与"接管"的组合，不是单纯关掉 |
| 命名插槽 + **插槽默认内容** | `<slot name="header">Default header content</slot>`，页面不给就用它 |
| 页面里的 `<NuxtLayout>` 别当根 | 同一条 important："If you use `<NuxtLayout>` within your pages, **make sure it is not the root element** (or disable layout/page transitions)."（示例外面确实套了 `<div>`） |

**完全不要壳的话就停在这儿**：`definePageMeta({ layout: false })`，页面模板直接写内容。

## 第三种：路由规则里 `appLayout: false`（v4.3）

layouts 目录页给的是三条并列规则，`false` 也在其中：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Set layout for specific route
    '/admin': { appLayout: 'admin' },
    // Set layout for multiple routes
    '/dashboard/**': { appLayout: 'dashboard' },
    // Disable layout for a route
    '/landing': { appLayout: false },
  },
})
```

官方给这条路的动机句值得完整读，它说了两个别的入口做不到的场景：

> This is useful when you want to manage layouts centrally in your configuration rather than in each page file, or when you need to **apply layouts to routes that don't have corresponding page components** (such as catchall pages which might match many paths).

**第二点很实在**：catch-all 页面匹配很多路径（115），你没法"在每个页面里写 `definePageMeta`"，因为只有一个文件；而按 URL 模式配 `appLayout` 正好覆盖这种形状。

三条入口同时存在时的优先关系在 180 引过的那条链里：页面 `layout` 元数据 → 路由规则 `appLayout` → `'default'`。**所以某页写了 `layout: false` 时，配置里的 `appLayout` 不会盖过它**。

## 第四种：写错名字，等于没布局（而且是静默的）

这条不是设计出来的关法，但它是我实际遇到过的一次"页面突然没壳"：

> `fallback`: If an invalid layout is passed to the `name` prop, **no layout will be rendered**. Specify a `fallback` layout to be rendered in this scenario. It **must** match the name of the corresponding layout file in the `app/layouts/` directory.
>
> - **type**: `string`
> - **default**: `null`

`fallback` 默认 `null`，所以"名字无效"的默认行为就是**什么都不渲染**。而名字最容易错的点不是拼写，是**归一化**：

> Please note the layout name is normalized to kebab-case, so if your layout file is named `errorLayout.vue`, it will become `error-layout` when passed as a `name` property to `<NuxtLayout />`.

**两种补救**：给 `fallback`（保底有壳），或者让声明式与文件名一致（layouts 目录页也推荐了这点：文件名匹配布局名）。

## 怎么确认它"真的没套"

三种关法都会让 `useLayout()` 读到同一个信号，官方对返回值的定义是：

> A read-only computed ref resolving to the layout name (a `string`), **or `false` when the layout is disabled**.

用法官方给的是页面外的例子（`app.vue`），正好可以当检查器：

```vue [app.vue]
<script setup lang="ts">
const layout = useLayout()
</script>

<template>
  <div>
    <CommandPalette v-if="layout !== 'minimal'" />
    <NuxtLayout>
      <NuxtPage />
    </NuxtLayout>
  </div>
</template>
```

那句关于作用范围的话也一并值得记：

> Within a rendered `<NuxtLayout>` it reflects **the enclosing layout**; outside of one (for example in `app.vue`) it returns **the layout that would be resolved for the current route**.

**注意它对第四种情况没有承诺**——`useLayout` 文档写的是"解析出的布局名或 `false`"，而"名字无效导致什么都不渲染"发生在 `<NuxtLayout>` 内部。这两件事是不是同一个返回值，我没在文档里找到答案，需要用一次断言式检查确认：故意传一个不存在的名字，看 `useLayout()` 给你字符串还是 `false`。

## 别忘第二类宿主：错误页

`<NuxtLayout>` 的定位句把两处并列写了出来：

> Nuxt provides the `<NuxtLayout>` component to show layouts **on pages and error pages**.

> You can use `<NuxtLayout />` component to activate the `default` layout on `app.vue` **or `error.vue`**.

官方给错误页的示例用的是 kebab 后的名字：

```vue [error.vue]
<template>
  <NuxtLayout name="error-layout">
    <NuxtPage />
  </NuxtLayout>
</template>
```

**这一格常被抓错**：`error.vue` 是独立入口，它不经过 `app.vue` 那棵树。所以"给错误页复用主站外壳"只能在这里显式套 `<NuxtLayout>`（或 `layout: false` 之后自己套），而不是指望布局链生效。

## 一张对照表

| 我要 | 用 | 谁读得到 |
| --- | --- | --- |
| 关掉某一页的布局 | `definePageMeta({ layout: false })` | `useLayout()` → `false` |
| 关掉后自己套壳/传插槽 | `layout: false` + 页面内 `<NuxtLayout name="custom">`（外层要有别的根） | 同上，读到的仍是 `false` |
| 关掉某段 URL、且那页没有对应页面文件 | `routeRules: { '/x/**': { appLayout: false } }` | 同上 |
| 整站临时不套（很少要） | `app.vue` 里 `<NuxtLayout :name="false">` | — |
| 防手滑写错名字 | `fallback="default"` | — |
| 错误页要另一套壳 | `error.vue` 里 `<NuxtLayout name="error-layout">` | — |

## 自查

1. `name` 的三种取值形态是什么？为什么框架要支持响应式引用？举官方给的取值来源例子。
2. 四种"没有布局"的写法分别叫什么、写在哪？哪一种不是设计而是失误？
3. `fallback` 的默认值是什么？"名字无效"时的默认行为是什么？
4. kebab 归一在哪两处出现？各针对什么写法？
5. `layout: false` 之后页面自己套 `<NuxtLayout>` 时，官方那条结构建议是什么？为什么外面还有 `<div>`？
6. `appLayout` 比 `definePageMeta` 强在哪个场景？官方动机句里的第二种情况是什么？
7. 三条入口同时存在时谁赢？在哪一页的哪句话里？
8. 怎么用 `useLayout()` 验证关闭成功？它对第四种情况有承诺吗？你打算怎么测？
9. 错误页为什么必须显式套 `<NuxtLayout>`？

## 下一篇

182 讲布局的层级问题：官方文档里"在一个布局里再放一个 `<NuxtLayout>`"这种写法其实没有记载，被记载的是嵌套目录的命名规则、页面内套布局、以及布局里放 `<NuxtPage>` 的那份三文件形状。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-layout`
  - 原文（定位与 name）："Nuxt provides the `<NuxtLayout>` component to show layouts on pages and error pages."；"You can use `<NuxtLayout />` component to activate the `default` layout on `app.vue` or `error.vue`."；"`name`: Specify a layout name to be rendered, can be a string, reactive reference or a computed property. It must match the name of the corresponding layout file in the `app/layouts/` directory, or `false` to disable the layout." / "**type**: `string | false`" / "**default**: `default`"；示例 `const layout = 'custom'` + `<NuxtLayout :name="layout"><NuxtPage /></NuxtLayout>`；note "Please note the layout name is normalized to kebab-case, so if your layout file is named `errorLayout.vue`, it will become `error-layout` when passed as a `name` property to `<NuxtLayout />`." + `error.vue` 示例；"`fallback`: If an invalid layout is passed to the `name` prop, no layout will be rendered. Specify a `fallback` layout to be rendered in this scenario. It must match the name of the corresponding layout file in the `app/layouts/` directory." / "**type**: `string`" / "**default**: `null`"
- `https://nuxt.com/docs/4.x/directory-structure/app/layouts`
  - 原文（三种指定方式与关闭）："Set the `appLayout` property in route rules."；"If you only have a single layout in your application, we recommend using `app.vue` instead."；"You can directly override the default layout for all pages using the `name` property of `<NuxtLayout>`" + `// You might choose this based on an API call or logged-in status`；"Overriding a Layout on a Per-page Basis" 全段（"If you are using pages, you can take full control by setting `layout: false` and then using the `<NuxtLayout>` component within the page." + 两段示例：`definePageMeta({ layout: false })` 配 `<NuxtLayout name="custom">` 与 `<template #header>`；布局侧 `<slot name="header">Default header content</slot>` 与 `<main><slot /></main>`）；important "If you use `<NuxtLayout>` within your pages, make sure it is not the root element (or disable layout/page transitions)."；route rules 示例 `'/admin': { appLayout: 'admin' }`、`'/dashboard/**': { appLayout: 'dashboard' }`、`'/landing': { appLayout: false }`；tip "This is useful when you want to manage layouts centrally in your configuration rather than in each page file, or when you need to apply layouts to routes that don't have corresponding page components (such as catchall pages which might match many paths)."；note "The layout name is normalized to kebab-case, so `someLayout` becomes `some-layout`."；note "If no layout is specified, `app/layouts/default.vue` will be used."
- `https://nuxt.com/docs/4.x/api/composables/use-layout`
  - 原文："> useLayout returns the layout resolved for the current route."；"using the same chain as `<NuxtLayout>`: the page's `layout` meta first, then the `appLayout` set via route rules, then `'default'`"；"Within a rendered `<NuxtLayout>` it reflects the enclosing layout; outside of one (for example in `app.vue`) it returns the layout that would be resolved for the current route."；"A read-only computed ref resolving to the layout name (a `string`), or `false` when the layout is disabled."；示例 `const layout = useLayout()` + `<CommandPalette v-if="layout !== 'minimal'" />`
- `https://nuxt.com/docs/4.x/directory-structure/app/pages` 与 `https://nuxt.com/docs/4.x/getting-started/views`
  - 对照用（见 059、094、180）：`definePageMeta` 的 `layout` 字段；三文件示例（115 的 catch-all 与本页 `appLayout` 的适用场景相关）
