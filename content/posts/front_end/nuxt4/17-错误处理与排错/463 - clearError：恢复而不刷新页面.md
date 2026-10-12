---
url: /posts/1c678f2c544ab469a62733aed2fc7844/
title: clearError：恢复而不刷新页面
date: 2027-03-15T10:00:00+08:00
updated: 2027-03-15T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：全屏错误页一旦挂上去，用户唯一的出路是手动刷新或改地址栏——`clearError()` 就是把它摘下来的那个动作。官方给的定义是"clears all handled errors"，可用位置写明是**页面、组件与插件**，参数只有一个可选对象 `options?: { redirect?: string }`，用途是"导航到一个安全的页面"。机制句同样重要：错误是被 `useError()` 写进状态的，`clearError` **重置这份状态并带着传入的 options 调用 `app:error:cleared` 钩子**——也就是说，想在"错误被清掉"这一刻做点什么，订阅点是钩子而不是返回值。

categories:
  - 前端开发

tags:
  - nuxt4
  - clearError
  - 错误页
  - 钩子
  - 中间件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

接口挂了，用户看到全屏错误页。你希望他点一个"重试"就能回到正常页面，而不是按浏览器的刷新键——这个动作在 Nuxt 里是 `clearError()`。

这一篇讲它清的是什么、能在哪调、以及清掉之后框架替你做了什么。

## 定义与可用位置

参考页的定性句：

> The clearError composable clears all handled errors.

**handled**（已被处理的）这个词是官方用词，配合下一句一起看：

> Within your pages, components, and plugins, you can use `clearError` to clear all errors and redirect the user.

可用位置写死三处：**页面、组件、插件**。参考页没有把服务端路由列进去。

## 参数只有一个是可选的

```
- `options?: { redirect?: string }`
```

官方对它的说明：

> You can provide an optional path to redirect to (for example, if you want to navigate to a 'safe' page).

示例给的是两种形态：

```ts
// Without redirect
clearError()

// With redirect
clearError({ redirect: '/homepage' })
```

**不带参数就是"只清错误、留在原处"；带 `redirect` 就是"清掉并且跳去某个路径"**。参考页里"safe page"这个说法点出了用意：错误状态下留在当前页通常没意义，跳到一个不依赖出错数据的地方才有意义。

错误处理那一页给它的签名是 `function clearError (options?: { redirect?: string }): Promise<void>`——**它返回 Promise**，和 `redirect` 配合时官方在 `error.vue` 示例里的用法是这样：

```vue [error.vue]
<script setup lang="ts">
import type { NuxtError } from '#app'

const props = defineProps({
  error: Object as () => NuxtError,
})

const handleError = () => clearError({ redirect: '/' })
</script>

<template>
  <div>
    <h2>{{ error?.status }}</h2>
    <button @click="handleError">
      Clear errors
    </button>
  </div>
</template>
```

## 机制：它其实改了状态、还发了一个钩子

参考页最后一句是这一篇最该记住的部分：

> Errors are set in state using [`useError()`](https://nuxt.com/docs/4.x/api/composables/use-error) to create a reactive and SSR-friendly shared error state across components. The `clearError` composable will reset this state and calls the `app:error:cleared` hook with the provided options.

三个动作连在一起：**错误由 `useError()` 写进那份响应式共享状态**（461 讲过它）、**`clearError` 重置这份状态**、**并带着你传入的 options 调用 `app:error:cleared`**。

所以想让"错误被清掉"这件事触发别的逻辑，订阅点是钩子。官方在 `app:error` 与 `app:error:cleared` 两行给的条目是这样的：

| 钩子 | 参数 | 环境 | 官方说明 |
| --- | --- | --- | --- |
| `app:error` | `err` | Server & Client | Called when a fatal error occurs. |
| `app:error:cleared` | `{ redirect? }` | Server & Client | Called when a fatal error occurs. |

这里要指出文档一处明显的问题：**`app:error:cleared` 那行的说明句和上一行 `app:error` 一字不差**，都在说"致命错误发生时调用"。而 clear-error 那一页写的是"**清掉错误时**调用"。**两处描述互相矛盾，文档没有解释哪个才是准的**——按函数页的描述更合理（那是它自己的实现说明），但要知道这一条在钩子表里没有对应的正确描述。两行的参数与环境列（`{ redirect? }`、Server & Client）与 clear-error 页的机制句是对得上的。

顺带接上 462：`showError` 调的是 `app:error`，`clearError` 调的是 `app:error:cleared`——**一进一出两个钩子成对**，做错误上报时两边都要接。

## 清掉之后还有两个坑

**第一，插件不会自动重跑。** 错误处理页那条 `::important` 是这么写的：

> Make sure to check before using anything dependent on Nuxt plugins, such as `$route` or `useRouter`, as if a plugin threw an error, then it won't be re-run until you clear the error.

也就是说**"清掉错误"正是插件得以重跑的那个节点**——反过来读：在错误还在的时候，`$route`、`useRouter` 这类插件产物可能压根没准备好。

**第二，跳转不一定靠 `redirect` 完成。** 460 记过 `::note`：渲染错误页是一次**独立的页面加载**，所有中间件都会再跑一遍。用 `redirect` 走的是这条跳；而如果在中间件里靠 `useError()` 判断状态来放行，那是另一条路（461）。两者不要混：`redirect` 是 `clearError` 的参数，不是中间件的逻辑。

## 自查

- `clearError()` 清的是"已被处理的错误"，官方给的可用位置是页面、组件、插件。
- 唯一参数是可选的 `{ redirect?: string }`；不给就留在原处，给了就跳去一个安全页；返回值是 `Promise<void>`。
- 它的机制是三件事：重置 `useError()` 的状态、带 options 调 `app:error:cleared`、让抛过错的插件有机会重跑。
- 钩子表里 `app:error:cleared` 的说明句抄的是 `app:error` 那句，与函数页矛盾，文档未解释。
- `showError` 对应 `app:error`、`clearError` 对应 `app:error:cleared`，上报要接两个。

## 下一篇

下一篇 464 讲错误页文件本身：`error.vue` 放在应用源码目录里、和 `app.vue` 同级；官方对它有写明一条硬规定——**它不是路由，不要放进 `pages/`，也因此不该在里面用 `definePageMeta()`**；它能拿到的是唯一一个 prop `error`，但可以用 `NuxtLayout` 并指定布局名来复用布局。

## 本文依据

- `clearError` 参考页：定性句（clears all handled errors）、可用位置（pages, components, and plugins）、参数 `options?: { redirect?: string }` 与"'safe' page"说明句、两种示例（不带参数与 `clearError({ redirect: '/homepage' })`）、机制句（useError 写状态、reset this state、calls the `app:error:cleared` hook with the provided options）、源文件 `composables/error.ts`（`api/utils/clear-error`）
- 错误处理页：`clearError` 签名 `(): Promise<void>` 与可选 redirect、`error.vue` 里 `handleError = () => clearError({ redirect: '/' })` 的完整示例、`::important` 关于插件抛错后到清掉错误前不会重跑、`::note` 关于错误页是独立页面加载所以中间件会再跑一遍（`getting-started/error-handling`）
- 钩子表 `app:error`（参数 `err`、Server & Client）与 `app:error:cleared`（参数 `{ redirect? }`、Server & Client、说明句与上一行重复）（`api/advanced/hooks`）
