---
url: /posts/2afaa85590039e96acb856b460514de3/
title: 模板里一个标签怎么找到组件：四个入口与 Lazy、island 的真实位置
date: 2026-12-04T11:00:00+08:00
updated: 2026-12-04T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把"模板里写一个标签，Nuxt 从哪儿把它解析出来"收拢成四个入口，并把几个容易被当成"运行时开关"的字段放回它们真正作用的层。**四个入口**：① 目录自动注册——"**Nuxt automatically imports any components in this directory (along with components that are registered by any modules you may be using).**"；② `#components` 显式导入——"**You can also explicitly import components from `#components` if you want or need to bypass Nuxt's auto-importing functionality.**"；③ `resolveComponent`——"**If you want to use the Vue `<component :is=\"someComputedComponent\">` syntax, you need to use the `resolveComponent` helper provided by Vue or import the component directly from `#components` and pass it into `is` prop.**"，且官方用 `::important` 限定："**make sure not to insert anything but the name of the component, which must be a literal string and not be or contain a variable. The string is statically analyzed at the compilation step.**"；④ 全局注册（159）。**Lazy 那一层**："**To dynamically import a component (also known as lazy-loading a component) all you need to do is add the `Lazy` prefix to the component's name.**"，收益官方说得很克制："**which can be helpful for optimizing your JavaScript bundle size**"，并且紧接着泼了冷水："Lazy components are great for controlling the chunk sizes in your app, **but they don't always enhance runtime performance, as they still load eagerly unless conditionally rendered.**"**`prefetch`/`preload` 的真实位置**只有一句："**These properties (prefetch/preload) are used in production to configure how components with Lazy prefix are handled by webpack via its magic comments.**"——生产期、只作用于 `Lazy` 前缀组件、机制是 webpack 魔法注释；`isAsync` 则是"**loaded async (with a separate chunk) regardless of using Lazy prefix or not**"。**island 的真实位置**有两处：`addComponent({ island: true })`/"If enabled, registers component as islands"，以及官方给的默认扫描目录——"**By default, component islands are scanned from the `~/components/islands/` directory. So the `~/components/islands/MyIsland.vue` component could be rendered with `<NuxtIsland name=\"MyIsland\" />`**"；`.server.vue` 后缀的组件"**use `<NuxtIsland>` under the hood, meaning that `lazy` prop and `#fallback` slot are both passed down to it**"，且"**Server components (and islands) must have a single root element. (HTML comments are considered elements as well.)**"`<NuxtIsland>` 自己的 `lazy` 定义是"**Make the component non-blocking.**"（默认 `false`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 组件解析
  - Lazy
  - 岛屿
  - 动态组件

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`<LazyMountainsList />` 这一行里同时藏着三件事：名字怎么来的、`Lazy` 前缀做了什么、它和 `island`/`prefetch` 这些字段是什么关系。前面几篇把入口讲散了，这一篇并成一张图。

## 一个标签的四个入口

| 入口 | 官方原话 | 什么时候用 |
| --- | --- | --- |
| ① 目录自动注册 | "Nuxt automatically imports any components in this directory (along with components that are registered by any modules you may be using)." | 常态 |
| ② `#components` 显式导入 | "You can also explicitly import components from `#components` if you want or need to **bypass Nuxt's auto-importing functionality**." | 需要组件对象、或发布模块（154） |
| ③ `resolveComponent` | "you need to use the `resolveComponent` helper provided by Vue **or** import the component directly from `#components` and pass it into `is` prop." | `<component :is="...">` |
| ④ 全局注册 | "register all your components globally... make them available throughout your application" | 159 讨论过代价 |

②和③官方给的例子是同一段：

```vue [app/pages/index.vue]
<script setup lang="ts">
import { SomeComponent } from '#components'

const MyButton = resolveComponent('MyButton')
</script>

<template>
  <component :is="clickable ? MyButton : 'div'" />
  <component :is="SomeComponent" />
</template>
```

两个细节都在语法层：`:is` 可以接受**字符串**（`'div'` 是原生元素）也可以接受**组件对象**；`resolveComponent` 的返回值与 `#components` 的导入在这里等价。

### `resolveComponent` 的参数字面量限制

这一条是硬的，官方专门用 `::important` 写：

> If you are using `resolveComponent` to handle dynamic components, make sure **not to insert anything but the name of the component, which must be a literal string and not be or contain a variable**. **The string is statically analyzed at the compilation step.**

**"编译期静态分析"这半句解释了症状**：如果名字来自变量或拼接，编译器分析不出来，运行时就是解析不到组件。想按运行时数据切换组件，就走另一条：把候选组件从 `#components` 导进来，再用对象映射去选。

（顺带一条内置组件的事实："There are a number of components that Nuxt provides, including `<ClientOnly>` and `<DevOnly>`."，它们不属于你的目录，但同样能出现在模板里。）

## `Lazy` 前缀做的事，官方两句写完

> To dynamically import a component (also known as lazy-loading a component) all you need to do is **add the `Lazy` prefix to the component's name**. This is particularly useful if the component is not always needed.

> By using the `Lazy` prefix you can delay loading the component code until the right moment, which can be helpful for **optimizing your JavaScript bundle size**.

官方的用法示例（注意它同时用了 `v-if`）：

```vue [app/pages/index.vue]
<script setup lang="ts">
const show = ref(false)
</script>

<template>
  <div>
    <h1>Mountains</h1>
    <LazyMountainsList v-if="show" />
    <button v-if="!show" @click="show = true">
      Show List
    </button>
  </div>
</template>
```

然后是那句我认为最该被记住的冷水：

> Lazy components are great for controlling the chunk sizes in your app, **but they don't always enhance runtime performance, as they still load eagerly unless conditionally rendered.**

拆成人话：

| 情况 | 结果 |
| --- | --- |
| `<LazyX />` 无条件渲染 | chunk **照样立刻加载**，只是被切成单独一块 |
| `<LazyX v-if="show" />` | 到 `show` 为真时才需要那块代码——收益才成立 |

所以 `Lazy` 前缀解决的是**分包**，不是"晚点加载"这件事本身；后者还需要一个渲染条件。官方随后一节（延迟水合）自己就写了这个分工："**Delayed (or Lazy) Hydration** — Lazy components are great for controlling the chunk sizes... In order to optimize your app, you may want to delay the hydration of some components until they're visible"（那一套 `hydrate-*` 策略我从 165 起单独讲）。

`Lazy` 前缀在 `#components` 里也能显式导入：官方 Direct Imports 的例子就是 `import { LazyMountainsList, NuxtLink } from '#components'`——**前缀是名字的一部分，不是模板指令**。

## `prefetch` / `preload` / `isAsync` 作用在哪一层

这三个字段我在 160 列过，但"它们到底什么时候起作用"才是选型关键。官方定义一句就够：

> These properties (prefetch/preload) are used **in production** to configure **how components with `Lazy` prefix are handled by webpack via its magic comments**. Learn more on webpack documentation.

三个限定词各自排除掉一类误解：

| 限定 | 排除掉的误解 |
| --- | --- |
| "**in production**" | 不是 dev 期的行为开关，开发时看不到差别 |
| "**components with Lazy prefix**" | 只对 `Lazy` 组件生效——与是否全局、是否岛屿无关 |
| "**handled by webpack via its magic comments**" | 机制是打包器的魔法注释（`webpackMode`），不是运行时调度。**如果你用的不是 webpack，这句话的适用性需要自己核实**（构建器选择见 088） |

`isAsync` 的口径不同，它明确说了不看前缀：

> This flag indicates, component should be loaded async (**with a separate chunk**) **regardless of using Lazy prefix or not**.

**这三个字段都出现在 `addComponentsDir` 与 `addComponent` 的参数表里**——也就是说它们是"注册时给组件打的构建期标记"，而不是模板上能写的东西。这一层我踩过理解上的坑：`prefetch` 与 147 里 `<NuxtLink>` 的 `prefetch` prop 完全不是一回事，一个是打包提示，一个是导航预取。

## `island: true` 与 `.server.vue` 的真实位置

岛屿有两条注册路径，官方都写了。

**第一条：目录约定。** `<NuxtIsland>` 那一页给的默认扫描目录：

> By default, component islands are scanned from the `~/components/islands/` directory. So the `~/components/islands/MyIsland.vue` component could be rendered with `<NuxtIsland name="MyIsland" />`.

注意这里标签的写法：岛屿不是靠标签名自动解析的，而是把名字**作为 `name` prop 传给 `<NuxtIsland>`**（`name` 是必需字段："Name of the component to render."）。

**第二条：字段注册。** `addComponentsDir({ island: true })` / `addComponent({ island: true })`，官方定义分别是"If enabled, registers components as islands."与"If enabled, registers component as islands. You can read more about islands in `<NuxtIsland/>` component description."

**第三条其实不是"注册"，是后缀**：目录页的 `.server` 一节说得很明白——

> Server-only components use [`<NuxtIsland>`] under the hood, meaning that **`lazy` prop and `#fallback` slot are both passed down to it**.

而 `<NuxtIsland>` 的 `lazy` 定义是"**Make the component non-blocking.**"（默认 `false`）。这两句接起来就能理解 107 里说的"`.server` 组件天然是岛屿"：

> Standalone server components will always be rendered on the server, **also known as Islands components**.
> When their props update, this will result in **a network request** that will update the rendered HTML in-place.

一条必须记住的结构限制（服务端组件与岛屿共用）：

> Server components (and islands) **must have a single root element**. (HTML comments are considered elements as well.)

**括号里那句是重点**：一个 HTML 注释就算一个"元素"，所以"根上先写注释再写 div"这种写法会直接违反单根要求。

## 后缀与前缀速查（这一层的完整一张表）

| 写法 | 效果 | 官方限制句 |
| --- | --- | --- |
| `Lazy` 前缀 | 动态导入、单独 chunk | "still load eagerly unless conditionally rendered" |
| `.client.vue` | 只在客户端渲染 | "This feature only works with Nuxt auto-imports and `#components` imports. **Explicitly importing these components from their real paths does not convert them into client-only components.**" |
| `.client.vue` 的时机 | 挂载后才渲染 | "`.client` components are rendered only after being mounted. To access the rendered template using `onMounted()`, **add `await nextTick()`** in the callback of the `onMounted()` hook." |
| `.server.vue` | 服务端渲染（岛屿） | 单根元素；props 变化触发网络请求 |
| `.server` + `.client` 同名成对 | 服务端一份、客户端一份 | "the `.server` + `.client` components are two 'halves' of a component and can be used in advanced use cases for **separate implementations** of a component on server and client side" |
| `.global.vue` | 全局注册 | "each global component is rendered in a separate chunk"（159） |
| `components/global/` | 整目录全局 | 与 `dirs` 默认第一项对应（156） |
| `components/islands/` | 岛屿名单 | "By default, component islands are scanned from the `~/components/islands/` directory" |
| `(分组)` 目录 | 只分组、不改名 | 157、158 |

成对写法官方给的最小例子就是一对文件 + 一个标签：

```bash [Directory Structure]
-| components/
---| Comments.client.vue
---| Comments.server.vue
```

```vue [app/pages/example.vue]
<template>
  <div>
    <!-- this component will render Comments.server on the server then Comments.client once mounted in the browser -->
    <Comments />
  </div>
</template>
```

## 自查

1. 模板里一个标签能被解析出来的四个入口分别是什么？官方给每个入口的原话关键词是什么？
2. `resolveComponent` 的参数为什么不能是变量？官方用哪半句解释了原因？
3. `<LazyX />` 不带 `v-if` 时，官方说会发生什么？`Lazy` 前缀真正解决的是哪个问题？
4. `prefetch`/`preload` 的三个限定词分别排除了什么误解？它们与 `<NuxtLink>` 的 `prefetch` prop 有什么关系？
5. `isAsync` 与 `Lazy` 前缀的区别在哪一个词上？
6. 岛屿的三条注册路径分别是什么？`<NuxtIsland>` 用哪个字段指定组件、它的 `lazy` 定义怎么说？
7. `.server.vue` 组件把哪两个东西透传给了 `<NuxtIsland>`？
8. 服务端组件与岛屿的单根要求里，括号那句为什么关键？
9. `.client.vue` 的两个限制分别是什么？为什么显式按真实路径导入会让"只在客户端"失效？

## 下一篇

162 讲 `defineNuxtComponent`：官方为什么在非 SFC 组件里要求用它而不是 `defineComponent`，以及它额外支持的那几项配置。

## 本文依据

- `https://nuxt.com/docs/4.x/directory-structure/app/components`
  - 原文（四个入口）："Nuxt automatically imports any components in this directory (along with components that are registered by any modules you may be using)."；Direct Imports — "You can also explicitly import components from `#components` if you want or need to bypass Nuxt's auto-importing functionality." + `import { LazyMountainsList, NuxtLink } from '#components'`；Dynamic Components — "If you want to use the Vue `<component :is=\"someComputedComponent\">`{lang=\"vue\"} syntax, you need to use the `resolveComponent` helper provided by Vue or import the component directly from `#components` and pass it into `is` prop." + 示例 `<component :is=\"clickable ? MyButton : 'div'\" />`、`const MyButton = resolveComponent('MyButton')`；important "If you are using `resolveComponent` to handle dynamic components, make sure not to insert anything but the name of the component, which must be a literal string and not be or contain a variable. The string is statically analyzed at the compilation step."；"Alternatively, though not recommended, you can register all your components globally, which will create async chunks for all your components and make them available throughout your application."
  - 原文（Lazy 与延迟水合的分工）："To dynamically import a component (also known as lazy-loading a component) all you need to do is add the `Lazy` prefix to the component's name. This is particularly useful if the component is not always needed."；"By using the `Lazy` prefix you can delay loading the component code until the right moment, which can be helpful for optimizing your JavaScript bundle size."；示例 `<LazyMountainsList v-if="show" />` 与 `<button v-if="!show" @click="show = true">Show List</button>`；"Lazy components are great for controlling the chunk sizes in your app, but they don't always enhance runtime performance, as they still load eagerly unless conditionally rendered."；"In order to optimize your app, you may want to delay the hydration of some components until they're visible, or until the browser is done with more important tasks."
  - 原文（Client Components）："If a component is meant to be rendered only client-side, you can add the `.client` suffix to your component."；note "This feature only works with Nuxt auto-imports and `#components` imports. Explicitly importing these components from their real paths does not convert them into client-only components."；important "`.client` components are rendered only after being mounted. To access the rendered template using `onMounted()`, add `await nextTick()` in the callback of the `onMounted()` hook."
  - 原文（Server Components）："Standalone server components will always be rendered on the server, also known as Islands components."；"When their props update, this will result in a network request that will update the rendered HTML in-place."；"Register server-only components with the `.server` suffix and use them anywhere in your application automatically."；"Server-only components use `<NuxtIsland>` under the hood, meaning that `lazy` prop and `#fallback` slot are both passed down to it."；warning "Server components (and islands) must have a single root element. (HTML comments are considered elements as well.)"；"In this case, the `.server` + `.client` components are two 'halves' of a component and can be used in advanced use cases for separate implementations of a component on server and client side." + `Comments.client.vue`/`Comments.server.vue` 与 `<Comments />`
  - 原文（内置组件）："There are a number of components that Nuxt provides, including `<ClientOnly>` and `<DevOnly>`."
- `https://nuxt.com/docs/4.x/api/kit/components`
  - 原文（三个构建期字段）：`prefetch`/`preload` — "These properties (prefetch/preload) are used in production to configure how components with Lazy prefix are handled by webpack via its magic comments. Learn more on webpack documentation"；`isAsync` — "This flag indicates, component should be loaded async (with a separate chunk) regardless of using Lazy prefix or not."；`island` — "If enabled, registers components as islands. You can read more about islands in `<NuxtIsland/>` component description."（`addComponent` 侧同句为 "If enabled, registers component as islands."）
- `https://nuxt.com/docs/4.x/api/components/nuxt-island`
  - 原文（定位与 props）："Server only components use `<NuxtIsland>` under the hood"；"When rendering an island component, the content of the island component is static, thus no JS is downloaded client-side."；"Changing the island component props triggers a refetch of the island component to re-render it again."；props 表：`name`（required，"Name of the component to render."）、`lazy`（boolean，default `false`，"Make the component non-blocking."）、`props`（"Props to send to the component to render."）、`source`（"Remote source to call the island to render."）、`dangerouslyLoadClientComponents`（default `false`）；note "Remote islands need `experimental.componentIslands` to be `'local+remote'` in your `nuxt.config`."；note "By default, component islands are scanned from the `~/components/islands/` directory. So the `~/components/islands/MyIsland.vue` component could be rendered with `<NuxtIsland name=\"MyIsland\" />`."；note "Component props and context are sent as GET query parameters to enable caching..."
  - 原文（插槽）："Slots can be passed to an island component if declared. Every slot is interactive since the parent component is the one providing it."；"`#fallback`: Specify the content to be rendered before the island loads (if the component is lazy) or if `NuxtIsland` fails to fetch the component."（见 107）
