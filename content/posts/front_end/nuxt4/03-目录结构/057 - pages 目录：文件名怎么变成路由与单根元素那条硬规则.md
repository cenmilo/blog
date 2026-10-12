---
url: /posts/813b7690007a7e0adf31a4de775ee56e/
title: pages 目录：文件名怎么变成路由，以及单根元素那条硬规则
date: 2026-10-31T09:00:00+08:00
updated: 2026-10-31T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：pages 目录是 Nuxt 的基于文件的路由系统，官方给了两个容易忽略的性质：一是这个目录为了减小包体是可选的——只用 app.vue 时不会包含 vue-router，要强制启用可以设 pages: true 或提供一个 router.options.ts；二是页面文件必须只有一个根元素，HTML 注释也算一个元素，官方解释的后果很具体：服务端渲染或静态生成时内容能正常显示，但客户端导航到该路由时路由过渡会失败、页面不会被渲染，并给了 working.vue、bad-1.vue（顶部一行注释就够）与 bad-2.vue（三个并列 div）三个对照例子。文件映射规则本身很简单：app/pages/index.vue 映射到 / 路由，页面可以是 Nuxt 支持的任何有效扩展名（默认 .vue、.js、.jsx、.mjs、.ts、.tsx），用 app.vue 时要用 <NuxtPage /> 显示当前页面。导航方面官方指定用 <NuxtLink>（它随 Nuxt 提供、不必导入），编程式导航用 navigateTo 且必须 await 或在函数里 return 它的结果。

categories:
  - 前端开发

tags:
  - nuxt4
  - 路由
  - pages
  - 单根元素
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


025 我们只建了两个页面文件，`/about` 就通了。这一篇讲**这套文件→路由的映射规则本身**，以及那条**最容易在你已经写完代码之后才发现的硬规则**（单根元素）。

## 第一条性质：这个目录是可选的，而且影响包体

官方在页面正文之前就写了这条 note：

> 为了减小你的应用包体，**这个目录是可选的**，也就是说**只使用 `app.vue` 时不会包含 `vue-router`**。要强制启用 pages 系统，可以在 `nuxt.config` 里设 `pages: true`，或者提供一个 [`router.options.ts`](https://nuxt.com/docs/4.x/guide/recipes/custom-routing#using-routeroptions)。

**这句和 054 那条是同一件事的两个入口**（054 抄的是 app 页那句"如果 `app/pages/` 不存在，Nuxt 就不会包含 vue-router 依赖"）。**这里给出的是强制打开的办法：`pages: true`。**

什么时候需要它？054 我说"别随手建空的 pages 目录"，但反过来——**如果你现在还没有页面文件，却想让路由系统就位（比如准备写第一个页面），`pages: true` 就是这个开关。**

## 文件与扩展名：页面就是 Vue 组件

> 页面是 Vue 组件，可以拥有 Nuxt 支持的任何[有效扩展名]（**默认是 `.vue`、`.js`、`.jsx`、`.mjs`、`.ts` 或 `.tsx`**）。
>
> **Nuxt 会为你 `~/pages/` 目录里的每一个页面自动创建一条路由。**

官方给了三种等价写法（同一个 index 页面）：

```vue [app/pages/index.vue]
<template>
  <h1>Index page</h1>
</template>
```

```ts [pages/index.ts]
// https://vuejs.org/guide/extras/render-function.html
export default defineComponent({
  render () {
    return h('h1', 'Index page')
  },
})
```

```tsx [pages/index.tsx]
// https://vuejs.org/guide/extras/render-function.html#jsx-tsx
export default defineComponent({
  render () {
    return <h1>Index page</h1>
  },
})
```

然后是那条最基本的映射：

> `app/pages/index.vue` 文件将被映射到你应用的 **`/` 路由**。

以及 054 讲过的那个前提：

> 如果你使用 `app.vue`，**请确保使用 `<NuxtPage/>` 组件来显示当前页面**：

```vue [app/app.vue]
<template>
  <div>
    <!-- Markup shared across all pages, ex: NavBar -->
    <NuxtPage />
  </div>
</template>
```

**（这一条我 025 讲过一次：没接 `<NuxtPage />` 时，`app.vue` 渲染的是它自己。）**

## 那条硬规则：必须单一根元素

官方用的是加粗的 must：

> 页面**必须有单一根元素**，以便在各页面之间启用[路由过渡]。**HTML 注释也被视为一个元素。**

**"注释也算元素"这半句是这条规则里最阴的部分。** 官方接着给了后果说明：

> 这意味着：当这条路由被服务端渲染、或被静态生成时，**你能正确看到它的内容**；但**当你在客户端导航过程中进入这条路由时，路由之间的过渡会失败，你会看到那条路由没有被渲染出来。**

**这就是"本地点链接进去是空白、直接刷新或分享链接进来却正常"这个症状的官方解释。** 服务端渲染路径与客户端导航路径的检查点不同——**多根节点只在后者那里出问题**。

官方给的三个对照例子，全部照抄：

**能工作的：**

```vue [app/pages/working.vue]
<template>
  <div>
    <!-- This page correctly has only one single root element -->
    Page content
  </div>
</template>
```

**不能工作的（原因是一行注释）：**

```vue [app/pages/bad-1.vue]
<template>
  <!-- This page will not render when route changes during client side navigation, because of this comment -->
  <div>Page content</div>
</template>
```

**不能工作的（三个并列 `div`）：**

```vue [app/pages/bad-2.vue]
<template>
  <div>This page</div>
  <div>Has more than one root element</div>
  <div>And will not render when route changes during client side navigation</div>
</template>
```

**把 `bad-1` 与 `working` 摆在一起看，差别只有一处：注释在根元素**里面**还是在根元素**外面**。**

于是这条规则的可执行版本是：**模板最外层必须且只能有一个节点，注释要写在它里面。** 054 里那个 `app.vue` 的三节点示例不受这条约束（约束是"页面"的），而 056 讲的错误页则另有要求（布局要有单一根元素）。**这三处各按各的文档写。**

## 预渲染那条条件（tip）

官方在这一节挂了一条很容易漏的 tip：

> **只有在你没有禁用 `prerender.crawlLinks`、并且你的 `prerender.routes` 列表里至少有一个页面时**，页面才会被自动注册用于预渲染。详见预渲染指南。

**这条的意义：建了页面不等于会被预渲染。** 它与 038 那条"默认两个兜底页都是空壳"是同一族问题——静态产物的内容取决于你构建期的配置，而不是文件是否存在。

## 导航：`<NuxtLink>` 不用导入

> 要在你的应用页面之间导航，你应该使用 `<NuxtLink>` 组件。
>
> **这个组件随 Nuxt 提供，因此你不必像其它组件那样导入它。**

```vue
<template>
  <NuxtLink to="/">Home page</NuxtLink>
</template>
```

编程式导航官方指定 `navigateTo()`，并附了一条必须遵守的 note：

> **请始终对 `navigateTo` 使用 `await`，或者在函数里 `return` 它的结果。**

```vue
<script setup lang="ts">
const name = ref('')
const type = ref(1)

function navigate () {
  return navigateTo({
    path: '/search',
    query: {
      name: name.value,
      type: type.value,
    },
  })
}
</script>
```

**注意官方这个例子走的是"`return`"那一条**（在函数里把结果返回出去），而不是 `await`。**两条都合法，但不能既不 await 也不 return。**

## 多个 pages 目录：靠图层

> 默认情况下，你所有的页面都应在项目根的一个 `app/pages` 目录里。
>
> 不过你可以用 Nuxt 图层来创建你应用页面的分组：

```bash [Directory Structure]
-| some-app/
---| nuxt.config.ts
---| pages/
-----| app-page.vue
-| nuxt.config.ts
```

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: ['./some-app'],
})
```

**读法：图层里那个 `pages/`（注意是 `some-app/pages`，不是 `some-app/app/pages`）通过 `extends` 被并进主项目。** 图层的完整叠加规则在 068。

## 自查

- [ ] 只用 `app.vue` 时少了哪个依赖？怎么强制启用 pages 系统？
- [ ] 页面文件可以用哪些扩展名？（官方给的默认那六个）
- [ ] 多根元素在 SSR 下和客户端导航下的表现差别是什么？
- [ ] 一行 HTML 注释为什么能让页面不渲染？该写在哪才安全？
- [ ] 建了页面就一定被预渲染吗？官方给的条件是什么？
- [ ] `navigateTo` 的两种正确写法是什么？

## 下一篇

058 讲文件名里的方括号：动态段、用双层方括号才生效的可选段、`[...slug]` 的 catch-all，以及官方那条"命名父路由优先于嵌套动态路由"的匹配优先级。

## 本文依据

- `directory-structure/app/pages`（标题：Nuxt 提供基于文件的路由来创建 Web 应用的路由；note：为减小包体该目录可选、只用 app.vue 时不会包含 vue-router，要强制 pages 系统可在 nuxt.config 设 pages: true 或提供 router.options.ts；Usage：页面是 Vue 组件可有 Nuxt 支持的任何有效扩展名（默认 .vue、.js、.jsx、.mjs、.ts、.tsx）、Nuxt 会为 ~/pages/ 里每个页面自动创建路由、tip 页面只有在未禁用 prerender.crawlLinks 且 prerender.routes 列表里至少有一个页面时才自动注册用于预渲染、三种 index 写法（index.vue 模板、index.ts 用 defineComponent+render+h、index.tsx 用 JSX）、app/pages/index.vue 映射到 / 路由、使用 app.vue 时要确保用 <NuxtPage/> 显示当前页面并给出 div 包 <NuxtPage /> 的示例；Pages must have a single root element 段：页面必须有单一根元素以启用各页面间的路由过渡且 HTML 注释也被视为元素、后果是服务端渲染或静态生成时能正确看到内容但客户端导航进入该路由时过渡失败且该路由不会被渲染、三个示例 working.vue（注释在 div 内）/bad-1.vue（模板顶部一行注释导致客户端导航时不渲染）/bad-2.vue（三个并列 div）；Navigation：应使用 <NuxtLink> 组件、它随 Nuxt 提供因此不必像其它组件那样导入、示例 <NuxtLink to="/">Home page</NuxtLink>；Programmatic Navigation：可用 navigateTo() 工具方法编程式导航、note 请始终 await navigateTo 或在函数里 return 其结果、示例 navigate() 里 return navigateTo({path:'/search', query:{name, type}})；Multiple Pages Directories：默认所有页面都在项目根的 app/pages 一个目录里、可用 Nuxt Layers 创建页面分组并给出 some-app/nuxt.config.ts + some-app/pages/app-page.vue 与主 nuxt.config extends ['./some-app'] 的示例）
