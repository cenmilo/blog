---
url: /posts/80879c898f1318238eaf286927b25410/
title: 624 - Bridge 的新 Composition API：七个旧 composable 各有去处、两个 lazy 变体与一个 nuxt2Context
date: 2027-05-08T09:00:00+08:00
updated: 2027-05-08T09:00:00+08:00
author: cmdragon

summary:
  摘要：`bridge/nuxt3-compatible-api`（页标题是 New Composition API）用**七个 `##` 小节**逐个安置旧包的 API：**`ssrRef` 与 `shallowSsrRef` → `useState`**（必须自己给 key，且**只能用在 `defineNuxtPlugin` 定义的插件或组件实例里**，因为全局上下文会带来跨请求共享状态的危险）、**`ssrPromise` 直接移除且没有替代**、**`onGlobalSetup` 移除**（可在插件里用 `nuxtApp.hook('vue:setup', …)` 或在布局 `setup()` 里跑）、**`useStore` → `useNuxtApp().$store`**、**`useContext` 与 `withContext` → `useNuxtApp`**（附带一个 `nuxt2Context`，官方明写**建议不要直接用、因为它在 Nuxt 3 不存在**）、**`wrapProperty` 不再提供但给了等价实现**、**`useAsync` 与 `useFetch` → `useLazyAsyncData` 与 `useLazyFetch`**。最后还有一条与 Nitro 有关：`::warning` 写着 **`useLazyFetch` 必须先配好 Nitro**；而上一段 `::important` 提醒"名字相似但 API 完全不同，别在 composable 外面改别的变量"。

categories:
  - 前端开发

tags:
  - nuxt4
  - bridge
  - composition-api
  - useState
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

Bridge 文档里最容易做错的一处是取数：名字还是那几个，含义已经换了。这一节把旧包的每个 API 都指定了去处，其中有三个是"没有替代，自己想办法"。

## 开场：这一节存在的理由

> By migrating from `@nuxtjs/composition-api` to the Nuxt 3 compatible API, there will be less rewriting when migrating to Nuxt 3.

也就是：**先把这一节的替换做完，真上 3 时改动的量最小。**

## 七个去处逐个看

### ssrRef 与 shallowSsrRef → useState

> These two functions have been replaced with a new composable that works very similarly under the hood: `useState`.

两点差别原文写得很细：**必须自己提供 key**（旧版由 Nuxt 自动生成），以及**只能在 `defineNuxtPlugin` 定义的插件或组件实例里调用**——括号里那句理由是：**不能用在全局/环境上下文，因为跨请求共享状态有风险。**

```diff
- import { ssrRef } from '@nuxtjs/composition-api'

- const ref1 = ssrRef('initialData')
- const ref2 = ssrRef(() => 'factory function')
+ const ref1 = useState('ref1-key', () => 'initialData')
+ const ref2 = useState('ref2-key', () => 'factory function')
  // accessing the state
  console.log(ref1.value)
```

因为是**按 key 存的**，同一个 key 可以在多处访问到同一份状态。

### ssrPromise：移除，无替代

> This function has been removed, and you will need to find an alternative implementation if you were using it. If you have a use case for `ssrPromise`, please let us know via a discussion.

**这一条不给任何迁移路径**——自己实现，或者去开讨论。

### onGlobalSetup：移除，两条替代路

替代是 `useNuxtApp` 或 `useState`（写在 `defineNuxtPlugin` 里），也可以在布局的 `setup()` 中跑自定义代码。官方给的对应写法把"全局 setup"变成了一个钩子订阅：

```diff
- import { onGlobalSetup } from '@nuxtjs/composition-api'

- export default () => {
-   onGlobalSetup(() => {
+ export default defineNuxtPlugin((nuxtApp) => {
+   nuxtApp.hook('vue:setup', () => {
      // ...
    })
- }
+ })
```

### useStore → useNuxtApp().$store

```diff
- import { useStore } from '@nuxtjs/composition-api'
+ const { $store } = useNuxtApp()
```

**注意这是"从 `useNuxtApp()` 解构出 `$store`"**，而不是再提供一个 `useStore`。

### useContext 与 withContext → useNuxtApp

```diff
- import { useContext } from '@nuxtjs/composition-api'
+ const { $axios } = useNuxtApp()
```

配套的 `::note` 是这一节最该记住的一句：

> `useNuxtApp()` also provides a key called `nuxt2Context` which contains all the same properties you would normally access from Nuxt 2 context, but **it's advised *not* to use this directly, as it won't exist in Nuxt 3.**

> Instead, see if there is another way to access what you need. (If not, please raise a feature request or discussion.)

也就是说：**逃生舱口存在，但用它就是在给自己埋下一次迁移的债**；真找不到替代，官方给的动作是提 feature request 或讨论，而不是"先用着"。

### wrapProperty：不再提供，但给了实现

> This helper function is not provided any more but you can replace it with the following code:

```ts
import { computed, getCurrentInstance } from 'vue'

const wrapProperty = (property: string, makeComputed = true) => () => {
  const vm = getCurrentInstance().proxy
  return makeComputed ? computed(() => vm[property]) : vm[property]
}
```

**这段是可以直接抄进项目的**（这一节唯一"官方给实现"的一条）。

### useAsync 与 useFetch → 两个 lazy 变体

> These two composables can be replaced with `useLazyAsyncData` and `useLazyFetch` … Just like the previous `@nuxtjs/composition-api` composables, **these composables do not block route navigation on the client-side** (hence the 'lazy' part of the name).

两条警告连着看：

- `::important`：**"Note that the API is entirely different, despite similar sounding names. Importantly, you should not attempt to change the value of other variables outside the composable (as you may have been doing with the previous `useFetch`)."**
- `::warning`：**"The `useLazyFetch` must have been configured for Nitro."** —— 与上一篇文章那条前置同源：**没配 Nitro 就别指望 `useLazyFetch` 能用**。

两个 diff 把返回值形状的变化摊开了：

```diff
<script setup>
- import { useAsync } from '@nuxtjs/composition-api'
- const posts = useAsync(() => $fetch('/api/posts'))
+ const { data: posts } = useLazyAsyncData('posts', () => $fetch('/api/posts'))
+ // or, more simply!
+ const { data: posts } = useLazyFetch('/api/posts')
</script>
```

```diff
<script setup>
- import { useFetch } from '@nuxtjs/composition-api'
- const posts = ref([])
- const { fetch } = useFetch(() => { posts.value = await $fetch('/api/posts') })
+ const { data: posts, refresh } = useLazyAsyncData('posts', () => $fetch('/api/posts'))
+ // or, more simply!
+ const { data: posts, refresh } = useLazyFetch('/api/posts')
  function updatePosts() {
-   return fetch()
+   return refresh()
  }
</script>
```

三点必须改：**多了第一个参数 key**、**结果在 `data` 里**、**旧的 `fetch()` 调用变成 `refresh()`**。这也正好回答 Overview 那一节 `::warning` 的疑问——它说 `useAsyncData`／`useFetch` 在 Bridge 下不可用，而这一节给的可用形态是**两个 lazy 变体**。

## meta：三条路，别同时走

`### useMeta` 一节给三个选项：

1. **`useNuxt2Meta`**：与 `vue-meta` 兼容地操作 meta，**"will work in Nuxt Bridge (but not Nuxt 3)"**；可以传 computed 或 ref，值会响应式更新。`::note`：**别在同一个组件里同时用 `useNuxt2Meta()` 与 Options API 的 `head()`，行为可能不可预测。**
2. **`useHead`**：Bridge 提供的 Nuxt 3 兼容实现，**需要在配置里显式开 `bridge.meta: true`**；底层是 `@unhead/vue`（不是 `vue-meta`），并建议**不要与原生 `head()` 并用**（会冲突）。
3. 指向 Nuxt 3 侧 seo-meta 文档的链接。

这三条与 622 那篇讲的是同一套约束，**只是这一节从"旧 `useMeta` 怎么换"的角度再说一遍**。

## 最后两小节：显式导入与关自动导入

`### Explicit Imports`：**Nuxt 把每个自动导入都挂在 `#imports` 别名上**，需要写显式 import 时用它：

```vue
<script setup lang="ts">
import { computed, ref } from '#imports'

const count = ref(1)
const double = computed(() => count.value * 2)
</script>
```

`### Disabling Auto-imports`：`imports.autoImport: false` 会**完全关掉自动导入**，但 `#imports` 的显式导入仍然可用。

**这里有个查不到东西的位置要提醒**：`imports` 在配置参考里只有三个子项（`dirs`、`global`、`scan`），**`autoImport` 那一节根本没有条目**——想查它的类型与默认值，配置页查不到，这一节这类用法句是唯一的依据。

## 自查

- 七个去处：`ssrRef`/`shallowSsrRef`→`useState`（要给 key、限上下文）、`ssrPromise`→**无替代**、`onGlobalSetup`→插件里 `nuxtApp.hook('vue:setup')` 或布局 `setup()`、`useStore`→`useNuxtApp().$store`、`useContext`/`withContext`→`useNuxtApp`（`nuxt2Context` 不建议用）、`wrapProperty`→官方给实现、`useAsync`/`useFetch`→两个 lazy 变体。
- 取数三条必改：**加 key、读 `data`、`fetch()`→`refresh()`**；`useLazyFetch` 需 Nitro 已配。
- 别在 composable 外部改别的变量（旧的 `useFetch` 用法要放弃）。
- meta 三选一，且**任意两条都不要在同一组件并存**。
- `#imports` 让自动导入可显式化；`imports.autoImport: false` 完全关自动导入（**该键在配置参考无条目**）。

## 下一篇

下一篇 625 给整个 600-629 收口：把 migration 11 页与 bridge 10 页的**结构清点**摆出来——**每页有几个小节、有没有编号步骤**，顺带说明一件事：**官方从没给过"总共有多少条变更"这个数**，任何统计都得自己说清是按什么数的。

## 本文依据

- https://nuxt.com/docs/4.x/bridge/nuxt3-compatible-api（页标题 "New Composition API" 与开场句；七个 `##` 小节全部原文与代码块（`useState` 的两点差别与跨请求理由、`ssrPromise` 无替代、`onGlobalSetup` 的 `vue:setup` diff、`useStore`、`useContext`/`withContext` 与 `nuxt2Context` 的 `::note`、`wrapProperty` 实现、`useAsync`/`useFetch` 段含 `::important` 与 `::warning` 及两个 diff）；`### useMeta` 三种做法与两条 note/warning；`### Explicit Imports` 的 `#imports` 段；`### Disabling Auto-imports` 的 `imports.autoImport: false` 示例与说明）
- https://nuxt.com/docs/4.x/bridge/overview（`::warning` 点名 `useAsyncData` 与 `useFetch` 不可用，与这一节 lazy 变体互为补充）
- https://nuxt.com/docs/4.x/bridge/nitro（`useLazyFetch` 依赖的 Nitro 配置前置）
- https://nuxt.com/docs/4.x/api/nuxt-config（`## imports` 只有 `dirs`／`global`／`scan` 三个子项，用于说明 `autoImport` 在配置参考无条目）
