---
url: /posts/57cf4b9c2a5bc6c6db8aa9d4ef0fc9b4/
title: 056 - error.vue：它不是路由页，error 对象里有八个字段
date: 2026-10-30T11:00:00+08:00
updated: 2026-10-30T11:00:00+08:00
author: cmdragon

cover: https://images.unsplash.com/photo-1584824486509-112e4181ff6b?auto=format&fit=crop&w=1600&q=80&fm=jpg

summary: >-
  摘要：error.vue 用来覆盖默认的出错页并把错误漂亮地显示出来；官方给它的唯一 prop 是 error，而那个对象的字段在文档里被完整列出：status（number）、fatal（boolean）、unhandled（boolean）、可选的 statusText、data、cause，外加两个被标注为 legacy/deprecated 的等价字段 statusCode（等价于 status）与 statusMessage（等价于 statusText）。这一节最重要的一条是 note：虽然它被叫做"错误页"，但它不是一条路由，也不该放进你的 pages 目录；出于同样的原因，你不该在这个文件里使用 definePageMeta——话虽如此，你仍然可以在错误文件里使用布局，方法是借助 NuxtLayout 组件并指定布局的名字。还有一条容易踩的规则：如果你抛出的错误带有自定义字段，它们会丢失，应当把这些字段放进 data。官方给的抛出示例是 createError({ status: 404, statusText: 'Page Not Found', data: { myCustomField: true } })。

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

<img src="https://images.unsplash.com/photo-1584824486509-112e4181ff6b?auto=format&fit=crop&w=1600&q=80&fm=jpg" title="056 封面：error.vue" alt="屏幕上显示的错误提示页面" style="max-width:100%;border-radius:8px;"/>

出错页大概是最容易做错的一类文件。你以为它是页面，其实它连路由都不是。这节官方话不多，但每句都能帮你躲开一次线上事故。

先看它是什么。应用跑着跑着，运行时总会冒出些意料之外的错误，官方的说法是，这时候可以用 `error.vue` 覆盖默认的错误文件，把错误漂亮地显示出来。

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

代码没什么花样，几个细节值得留意。

`NuxtError` 从 `#app` 导入，015 讲过的那类内置别名，这里是同一个体系里的另一个前缀。

`error` 是 prop，不是组合式函数的返回值，官方在下面明确说了，错误页就这一个 prop。

模板里留了个 `<NuxtLink to="/">`，用户看到错误页，总得有路可走。

最容易踩的是那条 note。官方专门强调：`error.vue` 虽然叫错误页，但它不是一条路由，别放进 `~/pages/` 目录；同理，`definePageMeta` 在这里也用不了。布局倒是可以套的，只是得写明用哪一个，用 `NuxtLayout` 组件指定布局名。

必须指定名字这点别忽略。059 讲布局时会提到，不写 name 默认用 `default.vue`，错误页里不吃这个默认，想用哪个布局就写哪个。

026 说布局必须单一根元素，错误页一样适用，别在这儿破例。

`error` 对象一共八个字段，官方把接口整个列了出来：

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

一眼看过去，`status`、`fatal`、`unhandled` 是三个必有字段，数字、布尔、布尔。其中 `fatal` 我想多说一句——它区分错误是不是致命的，界面文案给不给、要不要放重试按钮，我觉得都该看它。官方在这儿只给了类型，没说拿它干什么，这是我自己加的用法。

`statusText`、`data`、`cause` 是可选的。

剩下两个是历史包袱，接口里的注释写得很直白：`statusCode` 是 `status` 的旧等价，`statusMessage` 是 `statusText` 的旧等价。所以新代码直接写 `status` 和 `statusText`；旧示例、第三方模块里见到另外两个名字，知道它们是兼容层留下的就行，别当成两样不同的信息。015 讲 ESM 时提过，文档命名会随版本漂移，这儿又是一处。

还有条实际项目里最容易吃亏的：你往错误对象上挂的自定义字段，会被直接丢掉。官方的解法是塞进 `data`。

```ts
throw createError({
  status: 404,
  statusText: 'Page Not Found',
  data: {
    myCustomField: true
  },
})
```

接口里没有的键，带不过来，就这么简单。想传业务信息——哪个字段校验失败、哪个订单号找不到——只能放进 `data`。它的类型是 `unknown`，读的那一侧记得自己收窄。

部署上还有个坑。默认情况下两个兜底页都是空壳，038 的部署篇提过。想让你的 `error.vue` 渲染进 `404.html`，得把 `experimental.prerenderErrorPages` 打开，构建时会把它服务端渲染进去。

不开这个开关，`error.vue` 写得再好看也白搭，用户看到的 `404.html` 还是空壳。本地预览漂亮、线上白屏，多半就是这儿。

## 自查

- [ ] `error.vue` 为什么不能放进 `app/pages/`？因此哪个宏不能用？
- [ ] 错误页里能用布局吗？官方要求的写法是什么？
- [ ] `NuxtError` 里哪三个字段是必有的？哪两个是 legacy 等价物？
- [ ] 你想把自定义字段带到错误页，正确做法是什么？它在接口里是什么类型？
- [ ] 静态站点里想让 404 用上你的 `error.vue`，需要哪一个开关？

## 下一篇

057 进 `pages/` 目录本身：文件名怎么变成路由、以及那条「每个页面必须有单一根元素」的硬规则从哪来。
