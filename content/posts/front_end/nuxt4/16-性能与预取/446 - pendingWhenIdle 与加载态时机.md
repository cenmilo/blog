---
url: /posts/5fbfbbdb2e264474db6953e476fde561/
title: pendingWhenIdle 与加载态时机
date: 2027-03-09T11:00:00+08:00
updated: 2027-03-09T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：骨架屏写 `v-if="pending"`，在某些写法下永远不出现。原因在 `pendingWhenIdle` 这个默认关闭的实验项上：`pending` 只在请求真的在飞的时候为 `true`，`status` 处于 `idle` 时它是 `false`——`{ immediate: false }` 和 `{ server: false }` 在服务端渲染时正是这种"该显示加载态却没有"的时刻。官方给的结论句很明确：**加载 UI 要区分 idle 和在飞，就用 `status`**，而不是去翻全局开关。这一篇把 `pending`/`status` 的四种组合列成表，再补上另一层加载态：`useLoadingIndicator` 的 `throttle`（默认 200 ms）与 `duration`（默认 2000 ms）决定了进度条什么时候出现、什么时候消失。

categories:
  - 前端开发

tags:
  - nuxt4
  - 加载态
  - pendingWhenIdle
  - useFetch
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

页面顶部有个按钮，点下去才开始取数。你写的是 `v-if="pending"` 显示骨架屏——点了之后先是一片空白，数据直接蹦出来，骨架屏一闪都没闪。

另一处也一样：某个页面配了 `{ server: false }`，服务端渲染那一次 HTML 里没有内容，也没有加载态。

这两件事不是 bug，是 `pending` 的定义决定的。

## 先分清两个返回值

`useAsyncData` / `useFetch` 返回的东西里，官方列明了哪些是 ref：

> `data`, `status`, `pending`, and `error` are Vue refs. Access their values with `.value` in `<script setup>`. `refresh`/`execute` and `clear` are plain functions.

`status` 的取值是四个，来自类型定义：

```ts
type AsyncDataRequestStatus = 'idle' | 'pending' | 'success' | 'error'
```

**`idle` 是"没在飞"**，`pending` 是"在飞"。所以 status 有四种状态，而 `pending` 只是一个布尔——它必然在某些 status 下是 `false`。

## pendingWhenIdle 管的就是 idle 那一段

官方这一节的三句话把规则写完了。

默认值那一段：

> When `pendingWhenIdle` is `false` (the default), `pending` is `true` while a request is in flight and matches `status === 'pending'`. While `status` is `idle`, `pending` stays `false`. You can see this with `{ immediate: false }`, or with `{ server: false }` during server rendering.

翻过来那一段：

> Set `pendingWhenIdle: true` so `pending` is also `true` when `status` is `idle` and no cached data is available. Use `status` when your loading UI needs to distinguish `idle` from an in-flight request.

合起来是一张四格表：

| `status` | `pendingWhenIdle: false`（默认） | `pendingWhenIdle: true` |
| --- | --- | --- |
| `idle`（还没开始飞） | `pending` 为 `false` | `pending` 为 `true`（前提是没有缓存数据可用） |
| `pending`（在飞） | `pending` 为 `true` | `pending` 为 `true` |
| `success` / `error` | `pending` 为 `false` | `pending` 为 `false` |

官方点名你会撞上 `idle` 的两种写法，正好对应开头那两个场景：

- **`{ immediate: false }`**：取数不自动执行，要手动 `execute()`。在你调用它之前，`status` 就是 `idle`。
- **`{ server: false }` 在服务端渲染时**：这一侧不发请求，`status` 也是 `idle`。

第二格那句"and no cached data is available"要留意：开了 `pendingWhenIdle` 之后，`idle` 变成 `true` 还有一个附加条件——**没有缓存数据可用**。缓存里有数据时，`idle` 段仍然不算在飞。

## 开关是全局的，判据才是局部的

配置写法只有一种，作用范围是整个应用：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    pendingWhenIdle: true,
  },
})
```

这一改动会重定义**项目里每一处** `useAsyncData` 和 `useFetch` 的 `pending` 语义（官方第一句就是"`pendingWhenIdle` controls the `pending` ref returned by `useAsyncData` and `useFetch`"）。

而官方给的替代方案是一句话："Use `status` when your loading UI needs to distinguish `idle` from an in-flight request."——按这句做，写法是这样：

```vue
<script setup lang="ts">
const { data, status, refresh } = useFetch('/api/report', { immediate: false })
</script>

<template>
  <button @click="refresh()">生成报表</button>

  <p v-if="status === 'idle'">还没开始，先给个提示</p>
  <p v-else-if="status === 'pending'">正在生成，上骨架屏</p>
  <div v-else>{{ data }}</div>
</template>
```

用 `status` 判的好处是它把 `idle` 和在飞分开显示，你不用为了"让骨架屏出来"去改全局默认值。

## 另一层加载态：顶部进度条

上面是页面内容区的加载态。全站那条进度条归 `useLoadingIndicator`，它有自己的时机参数，三个默认值都要记：

| 参数 | 默认 | 官方说明 |
| --- | --- | --- |
| `duration` | `2000` | 进度条走完的时长，毫秒 |
| `throttle` | `200` | **对"出现"和"隐藏"做节流**，毫秒 |
| `estimatedProgress` | 内置函数 | 官方描述是"接近 100% 时逐渐放缓"，可传自定义函数，入参是 `duration` 与已耗时，返回 0-100 |

`throttle` 这一项解释了为什么快请求看不到进度条：**200 ms 内完成的话，它根本不出现**。这不是漏了，是节流。

它挂的钩子是 `page:loading:start` 与 `page:loading:end`，返回三个状态（`isLoading`、`error`、`progress`，都是只读的 ShallowRef）和四个方法（`start`、`set`、`finish`、`clear`）。三个方法都接受 `{ force: true }`：`start` 和 `set` 用它跳过 interval 立即显示，`finish` 用它跳过复位前的那段 interval。`finish` 的行为是"把 progress 设到 100、停掉所有计时器，**500 ms 后复位**"，另带 `{ error: true }` 改变条子颜色并把 `error` 置真。

## 选哪个信号

| 你想显示的 | 用什么 |
| --- | --- |
| 这一段内容正在请求 | `status === 'pending'`，或默认的 `pending` |
| 还没触发请求、要先给操作提示 | `status === 'idle'`（默认 `pending` 在这里是 `false`） |
| 全站导航期间顶部一条进度 | `<NuxtLoadingIndicator>` 或 `useLoadingIndicator`，注意 `throttle` |
| 出错时换颜色 | `useLoadingIndicator` 的 `finish({ error: true })` |

## 自查

- `pending` 默认不覆盖 `idle` 段，`{ immediate: false }` 和 `{ server: false }` 的服务端渲染都会落在 `idle`。
- `pendingWhenIdle: true` 让 `idle` 也算在飞，但**有缓存数据时仍不算**；而且它是全局的，会改掉项目里每一处 `pending` 的含义。
- 要区分 idle 与在飞，官方的建议是用 `status`，四个取值 `idle`/`pending`/`success`/`error`。
- 顶部进度条是另一层：`throttle` 200 ms 决定它出不出现，`duration` 2000 ms 决定它走多快，`finish` 之后 500 ms 复位。

## 下一篇

下一篇 447 讲比首字节还早的一件事：`writeEarlyHints`。官方只给了一句"Enables writing of early hints when using node server"——限定条件是 node 服务端，而 `writeEarlyHints` 同时还是 h3 里一个要你自己从 `h3` 导入的函数名，两者不是一回事。

## 本文依据

- `pendingWhenIdle` 全节：控制对象句（"controls the `pending` ref returned by `useAsyncData` and `useFetch`"）、默认 `false` 时与 `status === 'idle'` 的规则句、点名的两种写法（`{ immediate: false }`、`{ server: false }` 在服务端渲染时）、设 `true` 后的行为与"no cached data is available"条件、"Use `status` when your loading UI needs to distinguish `idle` from an in-flight request."、配置示例（`guide/going-further/experimental-features`）
- `data`/`status`/`pending`/`error` 是 ref、`refresh`/`execute`/`clear` 是普通函数的 `::note`；`AsyncDataRequestStatus` 四值类型（`api/composables/use-async-data`）
- `useLoadingIndicator`：三个参数与默认值（`duration` 2000、`throttle` 200、`estimatedProgress` 的 back off 描述）、`isLoading`/`error`/`progress` 三个只读 ShallowRef、`start`/`set`/`finish`/`clear` 四方法与 `force`、`finish` 的 500 ms 复位与 `error: true`、`page:loading:start`/`page:loading:end` 两个钩子、`estimatedProgress` 默认公式示例（`api/composables/use-loading-indicator`）
