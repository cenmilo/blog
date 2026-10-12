---
url: /posts/79d69c672035b8454283848d6b69174e/
title: NuxtPage 的五个 prop、pageKey 与子页面嵌套
date: 2026-11-11T11:00:00+08:00
updated: 2026-11-11T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`<NuxtPage>` 官方列出的 prop 有五个：`name`（告诉 `<RouterView>` 去渲染匹配到的路由记录 components 选项里对应名字的那个组件，配合 `name@view.vue` 文件名约定）、`route`（已解析其所有组件的路由位置，类型 `RouteLocationNormalized`）、`pageKey`（控制 `NuxtPage` 组件何时重新渲染，类型为 `string` 或 `function`）、`transition`（为所有由该组件渲染的页面定义全局过渡，类型 `boolean` 或 `TransitionProps`）、`keepalive`（控制页面状态保留，类型 `boolean` 或 `KeepAliveProps`），而官方同时注明 Nuxt 会自动解析 `name` 和 `route`，所以这两个通常不用你写。嵌套的机制是：`pages/parent.vue` 与 `pages/parent/child.vue` 会生成带 `children` 的路由记录（名字为 `parent` 与 `parent-child`），而要显示 `child.vue`，你必须**在 `app/pages/parent.vue` 里插入 `<NuxtPage>`**。控制重渲染有两条等价路径——父页面里 `<NuxtPage :page-key="route => route.fullPath" />`，或在子页面里 `definePageMeta({ key: route => route.fullPath })`；如果传一个永不改变的 key（官方例子是 `page-key="static"`），`<NuxtPage>` 只会在首次挂载时渲染一次；官方还警告不要在这里使用 `$route` 对象，因为它会给 `<NuxtPage>` 用 `<Suspense>` 渲染页面的方式造成问题。v4.5 的命名视图允许一个路由渲染进父组件里的多个出口（`<NuxtPage name="sidebar" />` 对应 `child@sidebar.vue`），没有匹配命名视图的出口会留空，且 `definePageMeta` 只从默认路由文件读取。想拿到页面组件的 ref 要走 `ref.value.pageRef`；自定义 prop 会一路传到页面组件，没写 `defineProps` 时仍可以从页面的 `attrs` 里直接取到。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxtpage
  - pagekey
  - 嵌套路由
  - 命名视图

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一篇看了 `<NuxtPage>` 的内部结构。这一篇把它能接受的输入和它作为"子页面出口"的用法讲完——**嵌套页面、重渲染时机、拿页面实例、往下传参**，这四件事全在同一个组件的五个 prop 上。

## 五个 prop，但你通常只需要写两个

官方列出的清单：

| prop | 官方说明 | 类型 |
| --- | --- | --- |
| `name` | 告诉 `<RouterView>` 去渲染"匹配到的路由记录的 components 选项里对应名字的那个组件"；配合 `name@view.vue` 文件名约定 | `string` |
| `route` | 已解析其所有组件的路由位置 | `RouteLocationNormalized` |
| `pageKey` | 控制 `NuxtPage` 组件**何时重新渲染** | `string` 或 `function` |
| `transition` | 为所有由 `NuxtPage` 组件渲染的页面定义全局过渡 | `boolean` 或 `TransitionProps` |
| `keepalive` | 控制由 `NuxtPage` 组件渲染的页面的状态保留 | `boolean` 或 `KeepAliveProps` |

官方在这张表后面挂了一条提示：**Nuxt 会自动解析 `name` 和 `route`，通过扫描 `/pages` 目录里找到的所有 Vue 组件文件来渲染它们。**

所以实际会被你写出来的只有三个：`pageKey`、`transition`、`keepalive`（加上命名视图时的 `name`）。`route` 不用手动传。

## 嵌套：子页面出口还得你自己摆

官方定义："可以配合 `<NuxtPage>` 显示嵌套路由。"目录结构是：

```bash [Directory Structure]
-| pages/
---| parent/
-----| child.vue
---| parent.vue
```

这棵树会生成这样的路由记录——注意 `children` 和 `name` 的形态：

```js
[
  {
    path: '/parent',
    component: '~/pages/parent.vue',
    name: 'parent',
    children: [
      {
        path: 'child',
        component: '~/pages/parent/child.vue',
        name: 'parent-child',
      },
    ],
  },
]
```

**关键的一句**：官方原话是"要显示 `child.vue` 组件，你必须插入 `<NuxtPage>` 组件到 `app/pages/parent.vue` 内部"。

也就是说，`<NuxtPage>` 不只出现在 `app.vue` 里，它是"页面出口"这个概念，父页面想要子页面渲染进来就得自己放一个。少了它，`/parent/child` 的匹配照样成立、`parent.vue` 照样渲染，只是子内容**不会出现**——这就是"嵌套页面打开是空白"最常被误解的一处。

## `pageKey`：控制"什么时候重新渲染"

官方给了三条，方向各不相同。

**① 传一个永不改变的 key → 只渲染一次。** 官方原文："例如，如果你传入一个从不改变的 key，`<NuxtPage>` 组件将只被渲染一次——在它首次挂载时。"

```vue [app/app.vue]
<template>
  <NuxtPage page-key="static" />
</template>
```

**② 用当前路由算一个动态 key：**

```html
<NuxtPage :page-key="route => route.fullPath" />
```

紧接着是一条警告，指向 091 里那个底层结构：**不要在这里使用 `$route` 对象，因为它会给 `<NuxtPage>` 用 `<Suspense>` 渲染页面的方式造成问题**（官方原话：Don't use `$route` object here as it can cause problems with how `<NuxtPage>` renders pages with `<Suspense>`）。要动态 key 就用回调参数里的 `route`，别去引用那个全局的 `$route`。

**③ 两条等价路径。** 官方原文："如果你想更多地控制 `<NuxtPage>` 组件何时重新渲染（例如为了过渡），你可以通过 `pageKey` prop 传入一个字符串或函数，**或者**通过 `definePageMeta` 定义一个 `key` 值。"

写在父页面的出口上：

```vue {}[pages/parent.vue]
<template>
  <div>
    <h1>I am the parent view</h1>
    <NuxtPage :page-key="route => route.fullPath" />
  </div>
</template>
```

或者写在子页面自己身上：

```vue {}[pages/parent/child.vue]
<script setup lang="ts">
definePageMeta({
  key: route => route.fullPath,
})
</script>
```

两者的差别是归属方向，不是效果：前者由**出口**统一决定所有子页面什么时候重建，后者由**页面自己**声明。官方另有一句同义表述——`pageKey` 可以作为 `key` 值通过 `definePageMeta` 从 `/pages` 目录里 Vue 组件的 `<script>` 部分传入。

顺带把 `transition` 的坑连上：上一篇说过启用过渡时页面必须单根；而"过渡没触发"的另一半原因往往在这里——key 不变，组件就不重建，过渡自然没有可过渡的对象。

## 命名视图：一个路由渲染进多个出口（v4.5）

官方定义：**单个路由可以通过给每个出口一个 `name`、并为每个名提供一个兄弟页面文件，渲染进父组件里的多个 `<NuxtPage>` 出口。** 文件名约定是 `name@view.vue`：

```bash [Directory Structure]
-| pages/
---| parent/
-----| child.vue
-----| child@sidebar.vue
---| parent.vue
```

然后从父页面按名渲染每个出口：

```vue [pages/parent.vue]
<template>
  <div>
    <NuxtPage />
    <aside>
      <NuxtPage name="sidebar" />
    </aside>
  </div>
</template>
```

官方的结果描述值得整句记住：当用户导航到 `/parent/child` 时，`child.vue` 渲染进默认的 `<NuxtPage />`，而 `child@sidebar.vue` 渲染进 `<NuxtPage name="sidebar" />`；**没有匹配到命名视图的出口会留空。**

还有一条容易吃亏的限制：**`definePageMeta` 只从默认路由文件读取；在 `name@view.vue` 兄弟文件里声明的 meta 对路由没有任何影响。** 换句话说，侧栏那个文件里写 `definePageMeta` 是白写，路由级的元数据只能放在 `child.vue`。

`name` 这个 prop 的作用也在这里对上了：它就是把出口指向匹配记录里那个对应名字的组件。

## 拿页面实例：走 `pageRef`

官方说法是："要获取一个页面组件的 `ref`，通过 `ref.value.pageRef` 访问它。"

```vue [app/app.vue]
<script setup lang="ts">
const page = ref()

function logFoo () {
  page.value.pageRef.foo()
}
</script>

<template>
  <NuxtPage ref="page" />
</template>
```

```vue [my-page.vue]
<script setup lang="ts">
const foo = () => {
  console.log('foo method called')
}

defineExpose({
  foo,
})
</script>
```

中间的 `.pageRef` 不能省。091 里那张结构图解释了为什么：`ref` 拿到的是 `<NuxtPage>` 这个包装组件本身，真正的页面在它里面。

## 自定义 prop 会一路传到页面

官方原文："`<NuxtPage>` 还接受你可能需要继续向下传递的自定义 prop。"

```vue [app/app.vue]
<template>
  <NuxtPage :foobar="123" />
</template>
```

在页面组件里可以访问这个 `foobar`：

```vue [app/pages/page.vue]
<script setup lang="ts">
const props = defineProps<{ foobar: number }>()

console.log(props.foobar) // Outputs: 123
</script>
```

**没声明也拿得到**——官方补了后半句："如果你没有用 `defineProps` 定义这个 prop，向下传给 `NuxtPage` 的 prop 仍然可以直接从页面的 `attrs` 访问到："

```vue [app/pages/page.vue]
<script setup lang="ts">
const attrs = useAttrs()
console.log(attrs.foobar) // Outputs: 123
</script>
```

嵌套路由那节的官方示例正是这条用法：`<NuxtPage :foobar="123" />` 写在 `pages/parent.vue` 里，`pages/parent/child.vue` 用 `defineProps({ foobar: String })` 接住再 `console.log(props.foobar)`。

## 四条误判

| 现场 | 真正的原因 |
| --- | --- |
| `/parent/child` 打开只有父内容 | `parent.vue` 里没插 `<NuxtPage>` |
| 换了参数但页面没重建 | 出口上的 key 不随参数变化（`page-key="static"` 只会渲染一次） |
| 过渡死活不触发 | key 没变，或页面不是单根元素 |
| 侧栏视图里的 `definePageMeta` 不生效 | meta 只从默认路由文件读取；`name@view.vue` 里的声明对路由无影响 |

## 自查

1. `<NuxtPage>` 的五个 prop 分别是什么？官方说哪两个由 Nuxt 自动解析？
2. 要让 `child.vue` 显示出来，必须做什么？
3. 控制重渲染的两条等价路径分别写在哪里？
4. 为什么官方警告不要在 `page-key` 里用 `$route` 对象？
5. `pages/parent/child@sidebar.vue` 这个文件渲染进哪个出口？没匹配到命名视图的出口是什么状态？
6. 想调用页面组件里 `defineExpose` 出来的方法，`ref` 应该怎么取？

## 下一篇

093 讲一个空壳应用打开时看到的东西：`<NuxtWelcome>` 这个组件、它在项目初始化里的角色，以及为什么它不该留在你的生产页面里。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-page`
  - 原文（Props 清单）："`name`: tells `<RouterView>` to render the component with the corresponding name in the matched route record's components option. See Named Views for the `name@view.vue` filename convention."，type `string`
  - 原文："`route`: route location that has all of its components resolved."，type `RouteLocationNormalized`
  - 原文："`pageKey`: control when the `NuxtPage` component is re-rendered."，type `string` or `function`
  - 原文："`transition`: define global transitions for all pages rendered with the `NuxtPage` component."，type `boolean` or `TransitionProps`
  - 原文："`keepalive`: control state preservation of pages rendered with the `NuxtPage` component."，type `boolean` or `KeepAliveProps`
  - 原文（tip）："Nuxt automatically resolves the `name` and `route` by scanning and rendering all Vue component files found in the `/pages` directory."
  - 原文（Example）："For example, if you pass a key that never changes, the `<NuxtPage>` component will be rendered only once - when it is first mounted."；`<NuxtPage page-key="static" />`（文件标注 `app/app.vue`）
  - 原文："You can also use a dynamic key based on the current route"；`<NuxtPage :page-key="route => route.fullPath" />`
  - 原文（warning）："Don't use `$route` object here as it can cause problems with how `<NuxtPage>` renders pages with `<Suspense>`."
  - 原文："Alternatively, `pageKey` can be passed as a `key` value via `definePageMeta` from the `<script>` section of your Vue component in the `/pages` directory."；示例 `definePageMeta({ key: route => route.fullPath })`
  - 原文（Page's Ref）："To get the `ref` of a page component, access it through `ref.value.pageRef`"；示例含 `page.value.pageRef.foo()`、`<NuxtPage ref="page" />`、页面侧 `defineExpose({ foo })` 与 `console.log('foo method called')`
  - 原文（Custom Props）："`<NuxtPage>` also accepts custom props that you may need to pass further down the hierarchy."；"For example, in the below example, the value of `foobar` will be passed to the `NuxtPage` component and then to the page components."；`<NuxtPage :foobar="123" />`；`const props = defineProps<{ foobar: number }>()` 与 `console.log(props.foobar) // Outputs: 123`
  - 原文："If you have not defined the prop with `defineProps`, any props passed down to `NuxtPage` can still be accessed directly from the page `attrs`"；`const attrs = useAttrs()`、`console.log(attrs.foobar) // Outputs: 123`
- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 原文（Nested Routes）："It is possible to display nested routes with `<NuxtPage>`."
  - 原文目录树：`pages/parent/child.vue` 与 `pages/parent.vue`
  - 原文生成的路由记录：`path: '/parent'`、`component: '~/pages/parent.vue'`、`name: 'parent'`、`children: [{ path: 'child', component: '~/pages/parent/child.vue', name: 'parent-child' }]`
  - 原文："To display the `child.vue` component, you have to insert the `<NuxtPage>` component inside `app/pages/parent.vue`"；示例 `<h1>I am the parent view</h1>` 与 `<NuxtPage :foobar="123" />`
  - 原文子页面示例：`const props = defineProps({ foobar: String })`、`console.log(props.foobar)`
  - 原文（Child Route Keys）："If you want more control over when the `<NuxtPage>` component is re-rendered (for example, for transitions), you can either pass a string or function via the `pageKey` prop, or you can define a `key` value via `definePageMeta`"；示例 `<NuxtPage :page-key="route => route.fullPath" />` 与 `definePageMeta({ key: route => route.fullPath })`
  - 原文（Named Views，badge v4.5）："A single route can render into multiple `<NuxtPage>` outlets in a parent component by giving each outlet a `name` and providing a sibling page file for each name."；"Use the `name@view.vue` filename convention to declare a named view alongside the default route file"
  - 原文目录树：`child.vue` 与 `child@sidebar.vue` 并列在 `parent/` 下
  - 原文示例：`<NuxtPage />` 与 `<aside><NuxtPage name="sidebar" /></aside>`
  - 原文："When the user navigates to `/parent/child`, `child.vue` renders into the default `<NuxtPage />` and `child@sidebar.vue` renders into `<NuxtPage name="sidebar" />`. Outlets without a matching named view are left empty."
  - 原文（note）："`definePageMeta` is read from the default route file only. Meta declared inside a `name@view.vue` sibling has no effect on the route."
  - 原文（Pages 首段）："The `app/pages/index.vue` file will be mapped to the `/` route of your application."；"If you are using `app.vue`, make sure to use the `<NuxtPage/>` component to display the current page"，示例含 `<!-- Markup shared across all pages, ex: NavBar -->`
  - 原文："Pages **must have a single root element** to allow route transitions between pages. HTML comments are considered elements as well."
- `https://nuxt.com/docs/4.x/getting-started/views`
  - 原文（Layouts 示例）：`<NuxtLayout><NuxtPage /></NuxtLayout>`
