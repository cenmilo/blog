---
url: /posts/a255b5b7f2d5be2288a836aa558323f7/
title: 296 - usePreviewMode：带 token 的内容预览模式
date: 2027-01-18T11:00:00+08:00
updated: 2027-01-18T11:00:00+08:00
author: cmdragon

summary:
  摘要：内容管理系统常有"草稿只在预览里可见、上线用户看不到"的需求。usePreviewMode 就是这个内置组合式：默认只要 URL 带 preview=true 就进入预览态，并把 URL 里的 token 存进 state 供取数用；一旦检测到预览态，它会自动强制 useAsyncData/useFetch 重渲染预览内容。这一篇把三个可配项讲全——自定义开启判据 shouldEnable、改默认 state 的 getState（注意它是追加、别误覆盖）、onEnable/onDisable 回调（默认都是 refreshNuxtData）。还落一条容易踩的测试口径：预览模式要用 nuxt generate + nuxt preview 测，不是 nuxt dev。

categories:
  - 前端开发

tags:
  - nuxt4
  - usePreviewMode
  - 预览
  - token
  - 取数
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

先说场景，否则这个组合式没头没脑。你接了个 CMS，编辑想先看"还没发布的草稿"在正式站点上长什么样，但普通访客不能看到草稿。这就是"预览模式"：一个只有带着特定标记才进得去、进去后内容强制重取的模式。`usePreviewMode` 是 Nuxt 内置用来管这件事的。

## 它是什么、返回什么

官方定位：

> Preview mode allows you to see how your changes would be displayed on a live site without revealing them to users.

> If the composable detects preview mode it will automatically force any updates necessary for `useAsyncData` and `useFetch` to rerender preview content.

最简用法：

```ts
const { enabled, state } = usePreviewMode()
```

- `enabled`：当前是不是预览态（一个可读写响应式值——官方示例里直接 `enabled = false` 关它）。
- `state`：预览态下存的东西，默认会把 URL 里的 `token` 参数收进来（下一节讲）。

默认判据是 URL 查询参数：

> By default the `usePreviewMode` composable will enable preview mode if there is a `preview` param in url that is equal to `true` (for example, `http://localhost:3000?preview=true`).

也就是访问 `/some-page?preview=true` 就进预览态。

## 组合式怎么和取数配合

官方完整示例：一个页面只有预览态才渲染草稿内容，取数时把 `state.token` 当查询参数带上：

```vue [app/pages/some-page.vue]
<script setup>
const { enabled, state } = usePreviewMode()

const { data } = await useFetch('/api/preview', {
  query: {
    apiKey: state.token,
  },
})
</script>

<template>
  <div>
    Some base content
    <p v-if="enabled">
      Only preview content: {{ state.token }}
      <br>
      <button @click="enabled = false">
        disable preview mode
      </button>
    </p>
  </div>
</template>
```

三个点串起来：

- `v-if="enabled"` 让草稿内容只在预览态出现——普通访客 `enabled` 为假，看不到那段。
- `useFetch` 用 `state.token` 作 `apiKey`——预览接口靠这个 token 返回草稿/未发布内容。这就是"带鉴权的预览"：token 从 URL 进 `state`、再进请求。
- 点 `enabled = false` 当场退出预览。

这一层和会话（259）是同一个思路——敏感内容靠一个服务端认的 token 把关，`usePreviewMode` 负责把这个 token 从 URL 收进 `state` 并在预览态强制取数。

## 可配项一：自定义开启判据 shouldEnable

默认只认 `?preview=true`。想换成别的判据（比如自定义 query 名），用 `shouldEnable`：

```ts
export function useMyPreviewMode () {
  const route = useRoute()
  return usePreviewMode({
    shouldEnable: () => {
      return !!route.query.customPreview
    },
  })
}
```

官方还建议把它包成自己的组合式，让各处判据一致、避免写歪——"to keep options consistent across usages and prevent any errors"。

注意这里在 `shouldEnable` 闭包里用了 `useRoute()` 取 `route.query`——这是在 setup 顶层同步调用、组合式外面再包一层，属于合法上下文（295 四类位置）。别把它挪进一个异步回调里再用。

## 可配项二：改默认 state 用 getState

默认 `state` 里放的是 URL 的 `token`。想往里塞别的东西，用 `getState`：

```ts
const data1 = ref('data1')

const { enabled, state } = usePreviewMode({
  getState: (currentState) => {
    return { data1, data2: 'data2' }
  },
})
```

有一条 note 必须看到，因为它是个静默的坑：

> The `getState` function will append returned values to current state, so be careful not to accidentally overwrite important state.

`getState` 返回的对象是**追加**到现有 state 上——所以如果你返回的键名和默认 state 里的（比如 `token`）撞了，会把原来那个**覆盖掉**。改 state 前先想清楚键名，别无意把 token 冲没。

## 可配项三：onEnable 与 onDisable 回调

默认行为官方写明了：

> By default, when `usePreviewMode` is enabled, it will call `refreshNuxtData()` to re-fetch all data from the server.
> When preview mode is disabled, the composable will attach a callback to call `refreshNuxtData()` to run after a subsequent router navigation.

- 进入预览 → 自动 `refreshNuxtData()`（全站重取数据，把草稿取回来）。
- 退出预览 → 挂在"下一次路由导航后"再 `refreshNuxtData()`（把数据刷回正式版）。

想换成自己的动作（或加自己的动作），传 `onEnable`/`onDisable`：

```ts
const { enabled, state } = usePreviewMode({
  onEnable: () => {
    console.log('preview mode has been enabled')
  },
  onDisable: () => {
    console.log('preview mode has been disabled')
  },
})
```

`refreshNuxtData` 是 210-249 那一路的东西（240 讲过全站重取），这里只是预览模式默认拿它来触发内容刷新。

## 测试口径：别用 nuxt dev

最后一条硬提醒，官方专门放了一个 note，因为很多人在这白折腾：

> `usePreviewMode` should be tested locally with `nuxt generate` and then `nuxt preview` rather than `nuxt dev`. (The `preview command` is not related to preview mode.)

也就是：

- 要验预览模式，先 `npx nuxt generate` 生成静态站、再 `npx nuxt preview` 跑起来，然后访问页面加 `?preview=true`。
- 别用 `nuxt dev` 测——预览模式的行为在 dev 下测不出来。
- 括号那句也别误解：**命令 `nuxt preview`（预览产物）和 `usePreviewMode`（内容预览模式）是两回事**，只是中文都叫"预览"。官方特意点破，正因为容易混。

## 自查

- 默认进预览就是 URL 带 `?preview=true`；要换判据用 `shouldEnable`（闭包里读 route.query 得在合法上下文同步取）。
- `state` 默认收 URL 的 `token`；改它用 `getState`，但它**追加**会覆盖同名键，别把 token 冲掉。
- 进出预览默认各自触发 `refreshNuxtData()`（退出是下一次导航后）；要自定义传 `onEnable`/`onDisable`。
- 测试走 `nuxt generate` + `nuxt preview`，不要用 `nuxt dev`；`nuxt preview` 命令 ≠ 预览模式。

## 下一篇

下一篇 297 回到钩子：讲怎么在构建期注册钩子，官方给的入口正是 nuxt-app 那四类上下文里的两个——`nuxt.config` 的 hooks 节，与模块 setup 里的 `nuxt.hook(...)`。

## 本文依据

- usePreviewMode（`api/composables/use-preview-mode`）：定位句与"detects preview mode ... force any updates necessary for useAsyncData and useFetch to rerender preview content"；`const { enabled, state } = usePreviewMode()`；Custom enable（默认 `?preview=true`、`useMyPreviewMode` 的 `shouldEnable` 示例、"keep options consistent" 建议）；Modify default state（默认存 URL `token`、`getState` 示例、note "getState will append returned values to current state, so be careful not to accidentally overwrite important state"）；onEnable/onDisable（默认 enabled 调 `refreshNuxtData()`、disabled 在下一次路由导航后调 `refreshNuxtData()`、自定义示例）；`some-page.vue` 示例（`v-if="enabled"`、`useFetch('/api/preview',{query:{apiKey: state.token}})`、`enabled = false`）；测试 note（"should be tested locally with nuxt generate and then nuxt preview rather than nuxt dev"、"The preview command is not related to preview mode"）。
