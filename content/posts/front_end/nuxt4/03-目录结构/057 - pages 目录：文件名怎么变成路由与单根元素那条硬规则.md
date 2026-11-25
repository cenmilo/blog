---
url: /posts/813b7690007a7e0adf31a4de775ee56e/
title: 057 - pages 目录：文件名怎么变成路由，以及单根元素那条硬规则
date: 2026-10-31T09:00:00+08:00
updated: 2026-10-31T09:00:00+08:00
author: cmdragon

cover: https://images.unsplash.com/photo-1461749280684-dccba630e2f6?auto=format&fit=crop&w=1600&q=80&fm=jpg

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

<img src="https://images.unsplash.com/photo-1461749280684-dccba630e2f6?auto=format&fit=crop&w=1600&q=80&fm=jpg" title="057 封面：pages 目录" alt="屏幕上的代码编辑界面" style="max-width:100%;border-radius:8px;"/>

025 我们只建了两个页面文件，`/about` 就通了。这一篇讲这套文件到路由的映射规则本身，以及那条最容易在你已经写完代码之后才发现的硬规则，单根元素。

先说一个容易忽略的性质：pages 目录是可选的，而且影响包体。官方专门在页面正文前挂了条 note——为了减小应用包体，只用 `app.vue` 时不会打包 `vue-router`。想强制启用，在 `nuxt.config` 里设 `pages: true`，或者提供一个 `router.options.ts`。

这条和 054 是同一件事的两个入口。054 说的是 `app/pages/` 不存在时 Nuxt 不会打包 vue-router，这里给的是强制打开的开关，`pages: true`。

什么时候用得上？054 里我说别随手建空的 pages 目录，但反过来，页面文件还没写、路由系统想先就位——比如准备写第一个页面的时候——`pages: true` 就是干这个的。

再看文件和扩展名。页面就是 Vue 组件，Nuxt 支持的任何有效扩展名都行，默认六个：`.vue`、`.js`、`.jsx`、`.mjs`、`.ts`、`.tsx`。`~/pages/` 里的每个页面，Nuxt 都会自动生成一条路由。

同一个 index 页面，官方给了三种等价写法。

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

最基本的映射就一条：`app/pages/index.vue` 映射到你应用的 `/` 路由。

还有 054 讲过的前提：用 `app.vue` 的话，记得用 `<NuxtPage/>` 显示当前页面。

```vue [app/app.vue]
<template>
  <div>
    <!-- Markup shared across all pages, ex: NavBar -->
    <NuxtPage />
  </div>
</template>
```

025 我讲过一次：不接 `<NuxtPage />`，`app.vue` 渲染的是它自己。

然后是那条硬规则。官方用了加粗的 must：页面必须单一根元素，路由过渡才启得起来。HTML 注释也算一个元素。

注释也算元素，是这条规则里最阴的部分。官方把后果写得很具体：这条路由被服务端渲染或静态生成时，内容显示正常；可一旦在客户端导航中进入它，过渡失败，页面就是不出来。

本地点链接进去空白、直接刷新或分享链接进来却正常——症状的官方解释就是这句。服务端渲染和客户端导航的检查点不同，多根节点只在后者那里出问题。

官方给了三个对照例子，照抄。

能工作的。

```vue [app/pages/working.vue]
<template>
  <div>
    <!-- This page correctly has only one single root element -->
    Page content
  </div>
</template>
```

不能工作的，原因是一行注释。

```vue [app/pages/bad-1.vue]
<template>
  <!-- This page will not render when route changes during client side navigation, because of this comment -->
  <div>Page content</div>
</template>
```

不能工作的，三个并列 `div`。

```vue [app/pages/bad-2.vue]
<template>
  <div>This page</div>
  <div>Has more than one root element</div>
  <div>And will not render when route changes during client side navigation</div>
</template>
```

`bad-1` 和 `working` 摆在一起，差别只有一处：注释在根元素里面，还是根元素外面。

所以这条规则的可执行版本就一句：模板最外层必须且只能有一个节点，注释写在它里面。054 那个 `app.vue` 三节点示例不受这条约束——约束的是页面；056 的错误页另有要求，布局要有单一根元素。三处各按各的文档来。

还有条容易漏的 tip：只有没禁用 `prerender.crawlLinks`、且 `prerender.routes` 里至少有一个页面，你的页面才会被自动注册进预渲染。

换句话说，建了页面不等于会被预渲染。和 038 那条兜底页空壳是同一族问题：静态产物的内容取决于构建期配置，不是文件在不在。

页面之间导航，官方指定用 `<NuxtLink>`。它随 Nuxt 提供，不用像其它组件那样导入。

```vue
<template>
  <NuxtLink to="/">Home page</NuxtLink>
</template>
```

编程式导航用 `navigateTo()`，官方附了一条必须遵守的 note：要么 `await`，要么在函数里 `return` 它的结果。

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

官方这个例子走的是 return 那条路，在函数里把结果返回出去。两条都合法，但既不 `await` 也不 `return` 不行。

页面多了想分组，靠图层。默认所有页面都该在项目根的 `app/pages` 里，官方的说法是，可以用 Nuxt 图层创建页面分组。

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

注意图层里那个 `pages/` 是 `some-app/pages`，不是 `some-app/app/pages`，通过 `extends` 并进主项目。图层完整的叠加规则在 068。

## 自查

- [ ] 只用 `app.vue` 时少了哪个依赖？怎么强制启用 pages 系统？
- [ ] 页面文件可以用哪些扩展名？（官方给的默认那六个）
- [ ] 多根元素在 SSR 下和客户端导航下的表现差别是什么？
- [ ] 一行 HTML 注释为什么能让页面不渲染？该写在哪才安全？
- [ ] 建了页面就一定被预渲染吗？官方给的条件是什么？
- [ ] `navigateTo` 的两种正确写法是什么？

## 下一篇

058 讲文件名里的方括号：动态段、用双层方括号才生效的可选段、`[...slug]` 的 catch-all，以及官方那条「命名父路由优先于嵌套动态路由」的匹配优先级。
