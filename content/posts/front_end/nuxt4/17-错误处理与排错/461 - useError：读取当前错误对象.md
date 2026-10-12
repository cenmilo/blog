---
url: /posts/6e49892fed8860a6b174f66264ba3e7f/
title: useError：读取当前错误对象
date: 2027-03-14T11:00:00+08:00
updated: 2027-03-14T11:00:00+08:00
author: cmdragon

summary:
  摘要：`useError()` 是读"当前正在被处理的那个错误"的唯一入口：不收任何参数，返回一个 `Ref`，**在客户端与服务端都可用**，官方写它提供的是"a reactive, SSR-friendly error state across your app"，错误状态变化时会自动更新，没有错误时值是 `undefined`。要用它的一个典型场景官方写得很具体：渲染错误页是一次**独立的页面加载**，你注册的所有中间件都会再跑一遍，所以中间件里要用 `useError()` 来判断"这次是不是在为错误页跑"。至于这个对象到底有哪些字段，官方三处给的清单互不一致——这一篇逐条对出来。

categories:
  - 前端开发

tags:
  - nuxt4
  - useError
  - 错误对象
  - 中间件
  - SSR
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

错误页渲染出来了，你的中间件也在同一时刻跑第二次。这时候想知道"现在是在处理错误吗"，官方给的入口就是 `useError()`。

这一篇讲它给什么、什么时候用，以及一个必须自己确认的坑：**这个错误对象的字段清单，官方三处写得不一样**。

## 基本形状

用法一行：

```ts
const error = useError()
```

官方定性句是：

> The `useError` composable returns the global Nuxt error that is being handled and is available on both client and server. It provides a reactive, SSR-friendly error state across your app.

三个信息：**两端可用**、**是响应式的**、**是全应用范围那一个错误状态**。

参数一节只有一句：

> This composable does not take any parameters.

返回值一节写的是：

> Returns a `Ref` containing the current Nuxt error (or `undefined` if there is no error). The error object is reactive and will update automatically when the error state changes.

所以判空要写在 `.value` 上，官方示例就是这样：

```vue
<script setup lang="ts">
const error = useError()

if (error.value) {
  console.error('Nuxt error:', error.value)
}
</script>
```

这一段的写法值得注意：它是在 `<script setup>` 里**同步**读一次 `.value`。如果错误是之后才出现的，要靠响应式（模板里用 `error?.status` 这类），而不是这段 `if`。

类型签名官方给的是：

```ts
export const useError: () => Ref<NuxtError | undefined>
```

## 最该用它的一处：中间件里

这一句在错误处理那一页，是这一篇最实用的部分：

> Rendering an error page is an entirely separate page load, meaning any registered middleware will run again. You can use `useError` in middleware to check if an error is being handled.

两个后果连在一起：**错误页不是普通路由**（056 讲过它不在 `app/pages/` 里、不该当路由用），**但它是另一次完整加载**，所以全站中间件会再执行一次。如果中间件里有"没登录就跳走"这类逻辑，它会在错误页上再跑一次，可能把用户从错误页带走。

601 那批会展开中间件细节，这里只记一条：中间件里第一件事是问 `useError()` 有没有值。

同一页还有一条相邻的 `::important`，是同一个场景的第二个坑：

> Make sure to check before using anything dependent on Nuxt plugins, such as `$route` or `useRouter`, as if a plugin threw an error, then it won't be re-run until you clear the error.

**插件抛过错，它在你清掉错误之前不会重跑**——所以错误页里凡是依赖插件产物（`$router`、`$route`、插件 `provide` 的东西）的代码，用之前要确认。

## 字段清单：三处对不上

这是必须自己核实的一块。官方三个页面各给了一份形状：

| 出处 | 给的是 | 列出的字段 |
| --- | --- | --- |
| 错误处理页的 `useError` 签名 | `Ref<Error \| { url, status, statusText, message, description, data }>` | `url`、`status`、`statusText`、`message`、`description`、`data` |
| `useError` 参考页的接口 | `interface NuxtError<DataT = unknown>` | `status: number`、`statusText?: string`、`message: string`、`data?: DataT`、`cause?: unknown`、`fatal: boolean` |
| `error.vue` 那一页的接口 | `interface NuxtError`（错误页的 `error` prop） | `status`、`fatal`、**`unhandled`**、`statusText?`、`data?`、`cause?`、`statusCode`（注明是 `status` 的 legacy/deprecated 等价）、`statusMessage?`（同上，对应 `statusText`） |

三处的差异不是措辞不同，是**成员不同**：

- `url`、`description` 只在错误处理页的签名里出现。
- `message` 出现在前两处，**不在** `error.vue` 那一页的清单里。
- `unhandled` 只在 `error.vue` 那一页出现。
- `statusCode`/`statusMessage` 只在 `error.vue` 那一页出现，而且官方直接标了 legacy/deprecated 等价。
- `fatal` 与 `cause` 在第二、三处有，第一处没有。

**这三份清单没有任何一处解释为什么会不一样**（文档未说明）。所以别把任何一份当权威。可靠做法是在你自己的项目里确认：

1. 在 `error.vue` 或组件里打印一次：`console.log(Object.keys(useError().value))`，拿实际键清单。
2. 看生成的类型：从 `#app` 导入的 `NuxtError` 才是你项目编译时真正用的那个（`error.vue` 的示例里就有 `import type { NuxtError } from '#app'` 这一行）。

两条里第 2 条更硬：`error.vue` 那一页写"如果你有需要自定义字段的错误，它们会**丢失**，应该放进 `data`"——这条同时说明那个对象的键是按类型收敛的，**自定义字段不会自动带过去**。

```ts
throw createError({
  status: 404,
  statusText: 'Page Not Found',
  data: {
    myCustomField: true,
  },
})
```

要传自己的结构化信息，官方给的位置是 `data`（这一句在 467 讲 `parseErrorData` 时会接到）。

## 自查

- `useError()` 无参数、两端可用、返回 `Ref<NuxtError | undefined>`，值随错误状态自动更新。
- 中间件里第一件事是它：错误页是独立加载，全站中间件会再跑一遍。
- 用插件产物（`$router`/`$route`）前要确认——插件抛错后到清掉错误之前不会重跑。
- 字段清单三处不一致，以你项目里 `#app` 的 `NuxtError` 类型和实际打印结果为准，不要照抄任何一份。
- 自定义字段会丢，要放进 `data`。

## 下一篇

下一篇 462 讲两个动作的分工：`createError()` 是**用来 throw 的**、双端可用，服务端抛出会触发全屏错误页、客户端抛出默认是非致命的，要全屏得自己写 `fatal: true`；`showError()` 是直接调用它、任何时刻都能调（服务端限中间件、插件或 `setup()`），而官方对它的建议是"其实更推荐用 `throw createError()`"。

## 本文依据

- `useError` 参考页：定位句（global Nuxt error、available on both client and server、reactive SSR-friendly error state）、`const error = useError()` 用法、"This composable does not take any parameters"、返回值段（`Ref`、无错误时 `undefined`、自动更新）、`NuxtError<DataT = unknown>` 六字段接口与 `() => Ref<NuxtError | undefined>` 签名、`console.error` 示例、指向错误处理页的 read-more、源文件 `composables/error.ts`（`api/composables/use-error`）
- 错误处理页：`useError` 签名行（含 `url`、`description`）、两条 `::note`（错误页是独立页面加载所以中间件会再跑一遍、可在中间件里用 `useError`；Node 16 下在错误页设 cookie 会覆盖先前 cookie 且官方建议升级）、`::important` 关于依赖插件的内容要先检查、`clearError` 的可选 redirect（`getting-started/error-handling`）
- `error.vue` 那一页：错误页只有 `error` 一个 prop、八字段 `NuxtError` 清单（含 `unhandled` 与两个标注 legacy/deprecated 的等价字段）、"自定义字段会丢失、应放进 `data`"及其 `createError({ data: {...} })` 示例、`import type { NuxtError } from '#app'`（`directory-structure/app/error`）
