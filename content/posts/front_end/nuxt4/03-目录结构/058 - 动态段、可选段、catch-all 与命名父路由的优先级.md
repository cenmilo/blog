---
url: /posts/29fe2ecba921d34131f17448776b0cde/
title: 058 - 动态段、可选段、catch-all 与命名父路由的优先级
date: 2026-10-31T10:00:00+08:00
updated: 2026-10-31T10:00:00+08:00
author: cmdragon

summary:
  摘要：文件名里的方括号决定路由形态：放进单层方括号的内容会变成动态路由参数，一个文件名或目录名里可以混用多个参数与非动态文本（官方例子是 users-[group]/[id].vue，访问 /users-admins/123 渲染出 admins - 123）；要参数可选则必须用双层方括号，官方给的两种写法 ~/pages/[[slug]]/index.vue 与 ~/pages/[[slug]].vue 都会同时匹配 / 和 /test；catch-all 用 [...slug].vue 这种文件名，它会匹配该路径下的所有路由，访问 /hello/world 时 $route.params.slug 是 ["hello", "world"]。匹配优先级有一条官方明确写出的规则：命名父路由优先于嵌套动态路由——对 /foo/hello 这条路径，~/pages/foo.vue 优先于 ~/pages/foo/[slug].vue，要分别匹配 /foo 与 /foo/hello 应改用 ~/pages/foo/index.vue 加 ~/pages/foo/[slug].vue。嵌套路由靠父页面里再放一个 <NuxtPage> 显示子组件（官方生成的路由对象里父 name 是 parent、子是 parent-child），子页面重渲染时机可用 <NuxtPage :page-key> 或 definePageMeta 的 key 控制。v4.5 起还支持命名视图（name@view.vue 约定，没有对应命名视图的出口留空，且 definePageMeta 只从默认路由文件读取），路由分组则用圆括号目录名且不影响 URL，v4.3 起分组自动出现在 route.meta.groups 里。

categories:
  - 前端开发

tags:
  - nuxt4
  - 路由
  - 动态段
  - catch-all
  - 优先级

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

057 讲的是文件名等于路径。这一篇讲文件名里那几对方括号，单层是动态段、双层是可选段、三个点是 catch-all、圆括号是分组，不进 URL。最后一条优先级规则，是这段里最容易让人排查半天的东西。

动态段，方括号里的是参数。官方原话，如果你把任何内容放进方括号里，它会被转成一个动态路由参数。你可以在一个文件名或目录名里混用多个参数，甚至与非动态文本混着放。

官方给的目录结构正是混着放的形态。

```bash [Directory Structure]
-| pages/
---| index.vue
---| users-[group]/
-----| [id].vue
```

读法，`users-[group]` 这一段是固定前缀 `users-` 加一个参数 `group`，里面再套 `[id]`。

参数怎么读，官方给了两条路。

```vue [app/pages/users-[group]/[id].vue]
<template>
  <p>{{ $route.params.group }} - {{ $route.params.id }}</p>
</template>
```

官方说，访问 `/users-admins/123` 会渲染出 `<p>admins - 123</p>`。

```vue
<script setup lang="ts">
const route = useRoute()

if (route.params.group === 'admins' && !route.params.id) {
  console.log('Warning! Make sure user is authenticated!')
}
</script>
```

官方补了一句，如果你想用组合式 API 访问路由，那有一个全局的 `useRoute` 函数，让你能像在选项式 API 里用 `this.$route` 那样访问路由。

`$route.params.xxx`，模板里，与 `useRoute().params.xxx`，`<script setup>` 里，是同一份数据的两种拿法，017 讲过这类模板里省了前缀的现象。

可选段，必须用双层方括号。官方原话，如果你希望一个参数是可选的，你必须把它包在双层方括号里，例如 `~/pages/[[slug]]/index.vue` 或 `~/pages/[[slug]].vue` 都会同时匹配 `/` 与 `/test`。

这一句里的两个信息都很硬。

官方用的是 must enclose it in double square brackets，写成 `[slug]` 就不是可选的。

两种落法都算，目录形式 `[[slug]]/index.vue` 与文件形式 `[[slug]].vue`。

判据是这条路由要不要在没有这个参数时也能命中。例如首页与列表页共用一个页面，`/` 与 `/test` 都命中，就是官方这个例子给的语义。

catch-all，三个点。官方原话，如果你需要一条 catch-all 路由，就用一个像 `[...slug].vue` 这样的文件名来创建它。它会匹配该路径之下的所有路由。

```vue [app/pages/[...slug].vue]
<template>
  <p>{{ $route.params.slug }}</p>
</template>
```

官方说，访问 `/hello/world` 会渲染出 `<p>["hello", "world"]</p>`。

注意参数形状，catch-all 给你的是数组，官方这个输出就是 `["hello", "world"]`。所以处理它的时候别把它当字符串用，这是 `[slug]` 与 `[...slug]` 最实际的差别。

那条优先级规则，命名父路由赢过嵌套动态路由。这一条我建议在写目录结构前先读一遍，因为它会让看起来该匹配的页面不匹配。官方原话，命名父路由优先于嵌套动态路由。对于 `/foo/hello` 这条路由，`~/pages/foo.vue` 会优先于 `~/pages/foo/[slug].vue`。要用 `~/pages/foo/index.vue` 与 `~/pages/foo/[slug].vue`，才能分别以不同页面匹配 `/foo` 与 `/foo/hello`。

把这三行翻成决策表。你想要的是 `/foo` 一个页面、`/foo/hello` 落到参数页，别这么写，`foo.vue` 加 `foo/[slug].vue`，因为 `foo.vue` 会抢走 `/foo/hello`。应该这么写，`foo/index.vue` 加 `foo/[slug].vue`。

症状是，你明明写了 `foo/[slug].vue`，但访问 `/foo/hello` 渲染的是 `foo.vue`。按官方这句，这不是 bug，而是命名父路由的优先级。解法是把那个精确路径改写成目录里的 `index.vue`。

嵌套路由，父页面里要再放一个 `<NuxtPage>`。官方原话，可以用 `<NuxtPage>` 显示嵌套路由。

官方给的目录与它生成的路由对象，这份对象值得留着，名字怎么生成的全在里面。

```bash [Directory Structure]
-| pages/
---| parent/
-----| child.vue
---| parent.vue
```

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

两点要记，子路由的 `name` 是 `parent-child`，父名加短横加子段，子路由的 `path` 是相对片段 `'child'` 而不是 `/child`。

然后那句关键要求，要在 `app/pages/parent.vue` 里插入 `<NuxtPage>` 组件，才能显示 `child.vue` 组件。

```vue [pages/parent.vue]
<template>
  <div>
    <h1>I am the parent view</h1>
    <NuxtPage :foobar="123" />
  </div>
</template>
```

```vue [pages/parent/child.vue]
<script setup lang="ts">
const props = defineProps({
  foobar: String,
})

console.log(props.foobar)
</script>
```

父到子的传值方式就是给 `<NuxtPage>` 加 prop、子页面用 `defineProps` 收，025 埋过一次，这里是官方给的完整一对示例。

子路由的重渲染时机，`pageKey` 或 `key`。官方原话，如果你想更控制 `<NuxtPage>` 组件何时重新渲染，例如为了过渡，你可以通过 `pageKey` prop 传一个字符串或函数，或者用 `definePageMeta` 定义一个 `key` 值。

```vue [pages/parent.vue]
<template>
  <div>
    <h1>I am the parent view</h1>
    <NuxtPage :page-key="route => route.fullPath" />
  </div>
</template>
```

```vue [pages/parent/child.vue]
<script setup lang="ts">
definePageMeta({
  key: route => route.fullPath,
})
</script>
```

两条通道做同一件事，差别在于写在父组件的 prop 上还是子页面的元数据里。`key` 这个字段的更多影响，它跟页面复用、滚动行为的关系，在 014 与 definePageMeta 那几篇展开。

命名视图，v4.5，一个路由渲染进多个出口。官方给的机制说明是，一条路由可以通过给每个 `<NuxtPage>` 出口一个 `name`、并为每个名字提供一个同级页面文件，渲染进父组件里的多个出口。使用 `name@view.vue` 文件名约定来在默认路由文件旁边声明一个命名视图。

```bash [Directory Structure]
-| pages/
---| parent/
-----| child.vue
-----| child@sidebar.vue
---| parent.vue
```

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

官方说，当用户导航到 `/parent/child` 时，`child.vue` 渲染进默认的 `<NuxtPage />`，而 `child@sidebar.vue` 渲染进 `<NuxtPage name="sidebar" />`。没有对应命名视图的出口保持为空。

还有一条必须知道的限制，`definePageMeta` 只从默认路由文件读取。在 `name@view.vue` 同级文件里声明的元数据对这条路由没有任何影响。

最后这条很容易踩，你在侧栏那个文件里写了 `definePageMeta({ middleware: ... })`，它不会生效，要写进 `child.vue`。

路由分组，圆括号不进 URL。官方原话，某些情况下你可能想把一组路由归到一起，而又不影响基于文件的路由。为此你可以把文件放进一个用圆括号包起来的文件夹，`(` 与 `)`。

```bash [Directory structure]
-| pages/
---| index.vue
---| (marketing)/
-----| about.vue
-----| contact.vue
```

官方说，这会在你的应用里产生 `/`、`/about` 与 `/contact` 三个页面。`marketing` 这个分组对你的 URL 结构而言是被忽略的。

v4.3 起还能读到它，路由分组会自动出现在路由元数据里，作为 `route.meta.groups`。这让你能在组件里访问分组信息，用于条件逻辑、样式或其它目的。

```vue [pages/(marketing)/about.vue]
<script setup lang="ts">
const route = useRoute()

console.log(route.meta.groups) // Output: ['marketing']
</script>

<template>
  <div>
    <p v-if="route.meta.groups?.includes('marketing')">
      This is a marketing page
    </p>
  </div>
</template>
```

分组的作用就清楚了，它是目录组织手段，还能给中间件或布局按组施加东西，不是 URL 的一部分。官方给的读法是数组，`['marketing']`，所以判断用 `includes`。

## 自查

- [ ] `[[slug]].vue` 与 `[slug].vue` 命中的路径差别是什么？
- [ ] catch-all 给你的 `params.slug` 是什么形状？
- [ ] `/foo/hello` 到底会命中哪个文件？官方给的改写办法是什么？
- [ ] 嵌套路由里子页面的 `name` 是怎么生成的？要在父页面里放什么？
- [ ] `child@sidebar.vue` 里写的 `definePageMeta` 会生效吗？
- [ ] `(marketing)/about.vue` 的 URL 是什么？分组名还能在代码里读到吗？

## 下一篇

059 讲 `layouts/` 目录：布局名怎么从嵌套路径生成（`desktop/default.vue` → `desktop-default`）、为什么单一根元素这条要求在那儿出现，以及 v4.4 起给布局传 props 的三种写法。
