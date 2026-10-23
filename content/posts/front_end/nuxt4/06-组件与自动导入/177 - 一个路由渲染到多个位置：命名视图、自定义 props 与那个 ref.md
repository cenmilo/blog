---
url: /posts/03f3dec15e8a4e96d3ef618fa59c1bee/
title: 177 - 一个路由渲染到多个位置：命名视图、自定义 props 与那个 ref
date: 2026-12-10T09:00:00+08:00
updated: 2026-12-10T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：前面讲 `<NuxtPage>` 时只用了"一个出口"这个默认形状，官方其实给了三个还没说的东西。**命名视图**（v4.5）："A single route can render into **multiple `<NuxtPage>` outlets** in a parent component by giving each outlet a `name` and **providing a sibling page file for each name**."，文件名约定是 "**Use the `name@view.vue` filename convention to declare a named view alongside the default route file**"（官方目录树：`pages/parent/child.vue` + `pages/parent/child@sidebar.vue` + `pages/parent.vue`），父级用 `<NuxtPage />` 与 `<NuxtPage name="sidebar" />` 两个出口接住；行为句是"When the user navigates to `/parent/child`, `child.vue` renders into the default `<NuxtPage />` and `child@sidebar.vue` renders into `<NuxtPage name=\"sidebar\" />`. **Outlets without a matching named view are left empty.**"，紧接一条必须知道的后果：**"`definePageMeta` is read from the default route file only. Meta declared inside a `name@view.vue` sibling has no effect on the route.**"**props 与 name 的自动解析**：官方 tip 是"Nuxt automatically resolves the `name` and `route` by **scanning and rendering all Vue component files found in the `/pages` directory**."。**自定义 props 会一路透传到页面组件**：`<NuxtPage :foobar="123" />` → 页面里 `defineProps<{ foobar: number }>()` 能拿到；"If you have not defined the prop with `defineProps`, any props passed down to `NuxtPage` can still be accessed directly from the page `attrs`"（`useAttrs().foobar`）。**拿页面实例的 ref** 走 `ref.value.pageRef`，官方示例配合页面里的 `defineExpose({ foo })`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 命名视图
  - NuxtPage
  - props
  - 元数据

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


092 那篇讲完了 `<NuxtPage>` 的五个 prop，看着像到头了。真做一个"主内容 + 侧栏"的布局时才发现，`name` 那个 prop 背后是一整套文件名约定，而它的元数据规则会让 `definePageMeta` 静默失效。

## 一个路由填多个出口

官方的定义句与目录约定（Named Views，标着 v4.5）：

> A single route can render into **multiple `<NuxtPage>` outlets** in a parent component by giving each outlet a `name` and providing a sibling page file for each name.

> Use the `name@view.vue` filename convention to declare a named view alongside the default route file:

```bash [Directory Structure]
-| pages/
---| parent/
------| child.vue
------| child@sidebar.vue
---| parent.vue
```

父组件里放两个出口：

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

行为与那个"空出口"规则：

> When the user navigates to `/parent/child`, `child.vue` renders into the default `<NuxtPage />` and `child@sidebar.vue` renders into `<NuxtPage name="sidebar" />`. **Outlets without a matching named view are left empty.**

**注意 URL 没变**：`/parent/child` 一个地址，两个组件位置。`@sidebar` 只存在于文件名里，不参与路径推导（对照 158 的 `(分组)` 目录——两者都是"文件名里的东西不影响 URL"，但一个决定**填哪个出口**，一个只是**不改名**）。

## 元数据只从默认文件读

这条是命名视图最容易踩的地方，官方单独用 note 写：

> `definePageMeta` is read from **the default route file only**. Meta declared inside a `name@view.vue` sibling has no effect on the route.

后果要摊开看：`child@sidebar.vue` 里写的 `layout`、`middleware`、`validate`、`keepalive`、`transition` **全部不生效**，因为编译器只把默认文件（`child.vue`）的元数据挂到这条路由记录上（117 讲的提取机制只认一个来源）。

所以两条实践规则：

1. **路由级的一切声明都写在默认文件里**（`child.vue`）。
2. 命名视图文件只负责渲染，需要行为就用它自己的 props/插槽，或干脆由默认文件统一声明。

**症状会是"我明明在 sidebar 里写了 `middleware: ['auth']`，但它没跑"**——这不是中间件的问题，是它根本没进路由记录。

## `name` 与 `route` 谁来填

props 表里这两行的定义是：

- `name`：tells `<RouterView>` to render the component with the corresponding name in the matched route record's components option. See Named Views for the `name@view.vue` filename convention.
- `route`：route location that has all of its components resolved.

官方用一条 tip 免掉了手动传参：

> Nuxt automatically resolves the `name` and `route` by **scanning and rendering all Vue component files found in the `/pages` directory**.

这解释了为什么绝大多数项目只写 `<NuxtPage />` 与 `<NuxtPage name="sidebar" />` 两种形态，`route` 基本不会手动给。

回到 091 那句根因：`<NuxtPage>` 是 `<RouterView>` 的包装，**应该用它而不是 `<RouterView>`**，否则 `useRoute()` 可能返回错误的路径。命名视图这套正是靠这层包装把 `name` 接进 vue-router 的 components 记录。

## 自定义 props 会一路传到页面

官方 "Custom Props" 一节给的是完整链路：

> `<NuxtPage>` also accepts custom props that you may need to pass further down the hierarchy. For example, in the below example, the value of `foobar` will be passed to the `NuxtPage` component and then to the page components.

```vue [app/app.vue]
<template>
  <NuxtPage :foobar="123" />
</template>
```

```vue [app/pages/page.vue]
<script setup lang="ts">
const props = defineProps<{ foobar: number }>()

console.log(props.foobar) // Outputs: 123
</script>
```

没声明也能拿到：

> If you have not defined the prop with `defineProps`, any props passed down to `NuxtPage` can still be accessed directly from the page `attrs`:

```vue [app/pages/page.vue]
<script setup lang="ts">
const attrs = useAttrs()
console.log(attrs.foobar) // Outputs: 123
</script>
```

**两种拿法的差别在 092/136 那条 props 规则上有对照**：`props: true` 是把路由 `params` 变成 props（vue-router 的机制），这里是把 `<NuxtPage>` 上的额外属性继续往下传。前者来自 URL，后者来自父组件模板——**用 `defineProps` 声明时两者名字别撞**，撞了之后靠 `attrs` 排查会很费劲。

## 拿到页面实例：pageRef

> To get the `ref` of a page component, access it through `ref.value.pageRef`

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

三个前提藏在例子里：

| 前提 | 依据 |
| --- | --- |
| 页面必须 `defineExpose` 那个方法 | 示例页面的 `defineExpose({ foo })` |
| 取的是 `ref.value.pageRef` 而不是 `ref.value` | 定位句 |
| 外层是 `<RouterView v-slot="{ Component }">` 那一串包装 | 091 的内部结构图（`<Transition>` → `<KeepAlive>` → `<Suspense>`） |

这与 176 提到的两个"ref 穿透"限制是同一族问题：**167 那条"Template refs 不能引用服务端组件内部元素"、以及 173 说的一些编译期特性只对 SFC 生效**——包装层数越多，`ref` 越要靠框架给的入口。

## 一张表：什么时候用哪个入口

| 我要 | 用 |
| --- | --- |
| 一个路由填两个位置（主内容 + 侧栏） | `name@view.vue` + `<NuxtPage name="…">`（元数据只写默认文件） |
| 页面按 URL 分段取参 | `props: true`（136） |
| 父级给当前页面传一个值 | `<NuxtPage :foobar="123" />` + `defineProps` 或 `useAttrs()` |
| 调用页面暴露的方法 | `<NuxtPage ref>` + `ref.value.pageRef.xxx()` + 页面里 `defineExpose` |
| 控制重建时机 | `page-key` / `definePageMeta({ key })`（137），别用 `$route` 对象（092 那条 warning） |
| 只想分组不改 URL | `(分组)` 目录（158） |

## 自查

1. 命名视图的两个条件是什么？官方给的文件名约定与目录树是什么形状？
2. 没有匹配命名视图的出口会怎样？`@sidebar` 会影响 URL 吗？
3. `definePageMeta` 从哪个文件读？在 `name@view.vue` 里写元数据会怎样、症状是什么？
4. `name` 与 `route` 两个 prop 通常不需要手动给，依据是哪句？
5. `<NuxtPage :foobar="123" />` 的值在页面里能用两种什么方式拿到？这与 `props: true` 的区别在哪？
6. 取页面实例的完整写法是什么？页面那一侧必须做什么？
7. 为什么 `<NuxtPage>` 不能换成 `<RouterView>`？

## 下一篇

178 讲一个只在模块场景下有意义的实验开关：`experimental.extraPageMetaExtractionKeys` 让构建期多读哪几个 key、为什么必须配 `pages:resolved` 用、以及官方要求模块同时增强 `NuxtPage` 类型。
