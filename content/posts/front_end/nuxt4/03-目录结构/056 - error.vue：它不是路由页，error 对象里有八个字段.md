---
url: /posts/57cf4b9c2a5bc6c6db8aa9d4ef0fc9b4/
title: error.vue：它不是路由页，error 对象里有八个字段
date: 2026-10-30T11:00:00+08:00
updated: 2026-10-30T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：error.vue 用来覆盖默认的出错页并把错误漂亮地显示出来；官方给它的唯一 prop 是 error，而那个对象的字段在文档里被完整列出：status（number）、fatal（boolean）、unhandled（boolean）、可选的 statusText、data、cause，外加两个被标注为 legacy/deprecated 的等价字段 statusCode（等价于 status）与 statusMessage（等价于 statusText）。这一页最重要的一条是 note：虽然它被叫做"错误页"，但它不是一条路由，也不该放进你的 pages 目录；出于同样的原因，你不该在这个文件里使用 definePageMeta——话虽如此，你仍然可以在错误文件里使用布局，方法是借助 NuxtLayout 组件并指定布局的名字。还有一条容易踩的规则：如果你抛出的错误带有自定义字段，它们会丢失，应当把这些字段放进 data。官方给的抛出示例是 createError({ status: 404, statusText: 'Page Not Found', data: { myCustomField: true } })。

categories:
  - 前端开发

tags:
  - nuxt4
  - 错误处理
  - error-vue
  - 布局
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


出错页是最容易被做坏的一类文件：**你以为它是页面，它其实不是路由。** 这一页官方给的话不多，但每一条都能避免一类线上事故。

## 它的定位与最小写法

> 在你应用的生命周期里，一些错误可能在运行时意外出现。**这种情况下，我们可以用 `error.vue` 文件来覆盖默认的错误文件，把错误漂亮地显示出来。**

```vue [error.vue]
<script setup lang="ts">
import type { NuxtError } from '#app'

const props = defineProps<{ error: NuxtError }>()
</script>

<template>
  <div>
    <h1>{{ error.status }}</h1>
    <NuxtLink to="/">Go back home</NuxtLink>
  </div>
</template>
```

三点值得注意：

1. **`NuxtError` 这个类型是从 `#app` 导入的**——015 讲过 `#build`/`#shared` 那类内置别名，这里是同一个体系里的另一个前缀。
2. **`error` 是 prop，不是组合式函数返回值。** 官方下面那句写得很明确：**"错误页有一个单一的 prop —— `error`"**。
3. 模板里给了一个出口：`<NuxtLink to="/">`。**用户看到错误页之后要有路可走。**

## 那条 note：它不是路由

这一页最重要的约束，官方用 note 写了三件事：

> 虽然它被叫做一个"错误页"，**但它不是一条路由，也不应该放进你的 `~/pages` 目录**。出于同样的原因，**你不应该在这个文件里使用 `definePageMeta`**。话虽如此，**你仍然可以在这个错误文件里使用布局，方法是利用 `NuxtLayout` 组件并指定布局的名字。**

拆成三条操作规则：

| 你想做的事 | 能不能 | 依据 |
| --- | --- | --- |
| 把 `error.vue` 放进 `app/pages/` | **不能** | 它不是路由 |
| 在里面写 `definePageMeta` | **不能** | 同上一条理由 |
| 在里面套一个布局 | **能**，但要显式指定布局名 | `<NuxtLayout name="...">` |

**"能套布局但必须指定名字"这半句很关键。** 059 会讲到没指定时用的是 `default.vue`；而这里官方说的是"utilizing the `NuxtLayout` component **and specifying the name of the layout**"——**在错误页里用布局要写明用哪一个。**

026 那条"布局必须单一根元素"的要求在这儿照样成立，别在错误页里破坏它。

## `error` 对象的八个字段

官方把这份接口整个列了出来，这是这一页信息密度最高的部分：

```ts
interface NuxtError {
  status: number
  fatal: boolean
  unhandled: boolean
  statusText?: string
  data?: unknown
  cause?: unknown
  // legacy/deprecated equivalent of `status`
  statusCode: number
  // legacy/deprecated equivalent of `statusText`
  statusMessage?: string
}
```

对着读，有三层信息：

**一、三个必有字段。** `status`（数字）、`fatal`（布尔）、`unhandled`（布尔）。

**`fatal` 值得单独想一下**：它区分"这个错误是不是致命的"——**界面文案与要不要给重试按钮，都该看它**（这是我给的使用建议；官方在这里只给了字段类型，没有说明该拿它做什么）。

**二、三个可选字段。** `statusText?`、`data?`、`cause?`。

**三、两个带 legacy/deprecated 注释的等价字段。** 官方在接口里就直接给了这两行注释：

```ts
// legacy/deprecated equivalent of `status`
statusCode: number
// legacy/deprecated equivalent of `statusText`
statusMessage?: string
```

**`statusCode` 是 `status` 的旧等价、`statusMessage` 是 `statusText` 的旧等价**——官方用注释直接标了 legacy/deprecated。

于是新旧两套字段的取舍很清楚：**新代码写 `status` 与 `statusText`。** 你会在旧示例、第三方模块里看到 `statusCode`/`statusMessage`，那是兼容层留下的名字。

（015 那一篇讲 ESM 时我说过一件事：文档里的命名会随版本漂移。**看到 `statusMessage` 不要以为它和 `statusText` 是两种不同的信息。**）

## 自定义字段会丢：官方给的解法是 `data`

这一条是实际项目里最容易踩的，官方用一句话 + 一个例子写清了：

> **如果你的错误带有自定义字段，它们会丢失；你应当把它们放进 `data`。**

```ts
throw createError({
  status: 404,
  statusText: 'Page Not Found',
  data: {
    myCustomField: true
  },
})
```

**判据很简单：`NuxtError` 那份接口里没有的键，不会被带过来。** 想传业务信息（比如"哪个字段校验失败""哪个订单号找不到"），**只能塞进 `data`**——而 `data` 的类型是 `unknown`，**读它的那一侧要自己做收窄**。

## 静态托管下这个文件谁来渲染

最后一条联系：038 抄过的部署页那段——

> 默认情况下，两个兜底页都是空壳。把 `experimental.prerenderErrorPages` 设为真，可**在构建时把你的 `error.vue` 服务端渲染进 `404.html`**。

**也就是说：`error.vue` 写得再好，静态部署时如果没开这一项，用户看到的 `404.html` 仍是空壳。** 这是"本地错误页漂亮、线上是白屏"的一种具体成因。

## 自查

- [ ] `error.vue` 为什么不能放进 `app/pages/`？因此哪个宏不能用？
- [ ] 错误页里能用布局吗？官方要求的写法是什么？
- [ ] `NuxtError` 里哪三个字段是必有的？哪两个是 legacy 等价物？
- [ ] 你想把自定义字段带到错误页，正确做法是什么？它在接口里是什么类型？
- [ ] 静态站点里想让 404 用上你的 `error.vue`，需要哪一个开关？

## 下一篇

057 进 `pages/` 目录本身：文件名怎么变成路由、以及那条"每个页面必须有单一根元素"的硬规则从哪来。

## 本文依据

- `directory-structure/app/error`（标题定义：error.vue 是你 Nuxt 应用的错误页；运行时可能意外出现错误，可用 error.vue 覆盖默认错误文件并把错误漂亮地显示出来；示例用 import type { NuxtError } from '#app' 与 defineProps<{ error: NuxtError }>()，模板渲染 error.status 并给出 <NuxtLink to="/">Go back home</NuxtLink>；note：虽然它被叫做错误页但它不是路由且不应放进 ~/pages 目录，出于同样原因不应在其中使用 definePageMeta，但仍可通过使用 NuxtLayout 组件并指定布局名来在错误文件里使用布局；"错误页有一个单一的 prop - error 其中包含待你处理的错误"；NuxtError 接口字段 status: number、fatal: boolean、unhandled: boolean、statusText?: string、data?: unknown、cause?: unknown、注释 legacy/deprecated equivalent of status 的 statusCode: number、注释 legacy/deprecated equivalent of statusText 的 statusMessage?: string；"如果你有一个带自定义字段的错误，它们会丢失；应当把它们放进 data"，示例 throw createError({ status: 404, statusText: 'Page Not Found', data: { myCustomField: true } })）
- `getting-started/deployment`（默认两个静态兜底页都是空壳；设 experimental.prerenderErrorPages 可在构建时把 error.vue 服务端渲染进 404.html）
- `directory-structure/app/layouts`（对照用：NuxtLayout 的 name 与默认 default.vue；布局必须有单一根元素）
- `api/nuxt-config`（别名体系与 #app 一类内置前缀的上下文，见 015/051）
