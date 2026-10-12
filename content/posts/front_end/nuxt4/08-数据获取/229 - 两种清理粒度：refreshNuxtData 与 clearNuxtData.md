---
url: /posts/41a7e96a7291a0145d2efcc6f38058b4/
title: 两种清理粒度：refreshNuxtData 与 clearNuxtData
date: 2026-12-27T10:00:00+08:00
updated: 2026-12-27T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一对函数名字很像，实际做的是两件不同的事，**一件重取、一件删除**。`refreshNuxtData` 的定位句是 "Refresh all or specific asyncData instances in Nuxt"，覆盖四个 composable（`useAsyncData`、`useLazyAsyncData`、`useFetch`、`useLazyFetch`），签名 `refreshNuxtData (keys?: string | string[])`，**不传参数时是全部重取**："All `useAsyncData` and `useFetch` keys are re-fetched when no `keys` are explicitly specified."，返回值是有意义的 Promise——"returns a promise, resolving when all or specific `asyncData` instances have been refreshed."。`clearNuxtData` 是删除："Delete cached data, error status and pending promises of useAsyncData and useFetch."，签名 `clearNuxtData (keys?: string | string[] | ((key: string) => boolean)): void`——**它比 refresh 多收一种参数（谓词函数），但返回 void，没有可等待的东西**；不传键时官方措辞是 "**all data** will be invalidated"。官方在两处给了优先级建议：**能拿到实例就用实例方法**——"If you have access to the `asyncData` instance, it is recommended to use its `refresh` or `execute` method as the preferred way to refetch the data."；另外 clear 那条 note 说明它的用途是"想让某个页面重新取数"。

categories:
  - 前端开发

tags:
  - nuxt4
  - refreshNuxtData
  - clearNuxtData
  - 缓存
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我用错一次的场景是这样的：退出登录后想让缓存消失，我调了 `refreshNuxtData()`，结果页面变成"加载中"然后拿到了旧数据（因为服务端接口那边还是老 session）。**那时我要的是删，不是重取。**

## 三档粒度，从上往下选

官方在 `refreshNuxtData` 页末尾给了明确倾向：

> If you have access to the `asyncData` instance, it is recommended to use its `refresh` or `execute` method as the preferred way to refetch the data.

所以我的选择顺序是：

| 粒度 | 写法 | 什么时候 |
| --- | --- | --- |
| 实例方法 | `await refresh()` / `await execute()` | 手上就有那次取数返回的对象（**官方推荐**） |
| 按 key | `await refreshNuxtData('todos')` 或 `await refreshNuxtData(['count', 'user'])` | 在别处（组件外、事件回调里）想重取某几格 |
| 全站 | `await refreshNuxtData()` | 几乎不该用——**它重取所有 key** |

三档的差别不是方便程度，是**请求量的量级**。我在列表页上写 `refreshNuxtData()` 那次，触发的是整页所有取数，包括用户根本没在看的那几个标签页数据。

## refreshNuxtData：两条示例都是"防重入"形状

签名与返回：

```ts [Signature]
export function refreshNuxtData (keys?: string | string[])
```

- 参数："A single string or an array of strings as `keys` that are used to fetch the data. This parameter is **optional**. All `useAsyncData` and `useFetch` keys are re-fetched when no `keys` are explicitly specified."
- 返回："`refreshNuxtData` returns a promise, resolving when all or specific `asyncData` instances have been refreshed."

官方两段示例给的是同一个 UI 形状（Refresh All 与 Refresh Specific），我觉得值得抄的是**按钮与状态的处理**：

```vue [app/pages/some-page.vue]
<script setup lang="ts">
const refreshing = ref(false)

async function refreshAll () {
  refreshing.value = true
  try {
    await refreshNuxtData()
  } finally {
    refreshing.value = false
  }
}
</script>

<template>
  <div>
    <button
      :disabled="refreshing"
      @click="refreshAll"
    >
      Refetch All Data
    </button>
  </div>
</template>
```

按 key 那一段只有一行不同，并留了一句注释说明数组也合法：

```ts
    // you could also pass an array of keys to refresh multiple data
    await refreshNuxtData(['count', 'user'])
```

这个 `try/finally` + `:disabled` 的组合解决的是同一件事：**刷新期间用户又点一次**。虽然实例上的 `refresh` 自己会等上一次跑完（214 那句 "By default, Nuxt waits until a `refresh` is finished before it can be executed again."），但按钮上的加载态还是得自己管。

还有一条 note 我觉得是这类"全局刷新"最容易忽略的副作用：

> If your component is cached by `<KeepAlive>` and enters a deactivated state, the `asyncData` inside the component will still be refetched until the component is unmounted.

也就是说 **`refreshNuxtData()` 会打到那些当前根本不显示的保活组件身上**（137 讲过 keepalive 保留状态；这条是它的代价）。

## clearNuxtData：删除，而且是三档输入

整页很短，全部内容都在签名和一句参数说明里：

```ts [Signature]
export function clearNuxtData (keys?: string | string[] | ((key: string) => boolean)): void
```

| 项目 | 官方原话 |
| --- | --- |
| 定位句 | "Delete cached data, **error status and pending promises** of useAsyncData and useFetch." |
| 参数 | "One or an array of keys that are used in `useAsyncData` to delete their cached data. If no keys are provided, **all data** will be invalidated." |
| 用途 note | "This method is useful if you want to invalidate the data fetching for another page." |

三点我要划出来：

1. **删三样**：缓存数据、错误状态、进行中的 promise。所以它也能清掉"卡住的错误态"，这是 `clear()` 实例方法（214 那四件事）之外更彻底的一档。
2. **接受谓词函数**：`(key: string) => boolean`。这是 `refreshNuxtData` 没有的能力——按前缀批量删是它的独有出口，官方没给例子，我按类型能推出的用法是形如 `clearNuxtData(key => key.startsWith('user:'))`（**这是我的推断写法，文档没有这种示例，用前实测一次**）。
3. **返回 `void`**：没有 Promise，**你 await 不到任何东西**。这一点和 refresh 相反，也解释了为什么退出登录后"删完立刻跳到列表页"能生效（新页面自己会重新取），而"删完等着它同步"这种需求根本不存在。

覆盖范围这里也写窄了一格：`clearNuxtData` 的说明只点 `useAsyncData` 和 `useFetch`，而 `refreshNuxtData` 的定位句列了四个（含两个 `useLazy*`）。**按字面理解，lazy 那两族在 clear 的说明里没被点名**——文档没解释这个差别，我不替它扩展。

## 退登时到底该调哪个

我把两者的语义并起来看，结论很直接：**退登要的是"这些缓存不再可信"，所以是删；重取只会拿到旧凭据下的结果。**

```ts
// 我的写法：删掉用户相关的格，再跳转
clearNuxtData(key => key.startsWith('user:'))
await navigateTo('/login', { external: false })
```

**这里的谓词用法沿用上面那条推断**，如果你的 key 命名不规则，就退回官方唯一给的形态：不传参数全清（`clearNuxtData()`）。代价是别的页数据也没了，下一次进页面会重新取。

至于 `serialize: false`（217）与 `payloadExtraction`（213）——它们决定"缓存本来有没有建立"，和这里的"事后清理"是不同层的东西，别拿清理函数去补配置缺位。

## 自查

1. 三个重取粒度里官方推荐哪个？依据句是什么？（答：实例的 `refresh`/`execute`；原话是 "If you have access to the asyncData instance, it is recommended to use its refresh or execute method as the preferred way to refetch the data."）
2. `clearNuxtData` 能删哪三样，返回什么？（答：缓存数据、错误状态、进行中的 promise；返回 `void`，不可 await。）
3. `clearNuxtData` 的参数比 `refreshNuxtData` 多一种什么形态？（答：谓词函数 `(key: string) => boolean`。）

## 下一篇

230 - callOnce：只跑一次那一档与 navigation 模式

## 本文依据

- `api/utils/refresh-nuxt-data`：定位句（覆盖四个 composable）；`<KeepAlive>` deactivated 那条 note；签名 `keys?: string | string[]`；Parameters 句（optional、不给键时全部重取）；Return Values 句（Promise 何时 resolve）；Refresh All Data 与 Refresh Specific Data 两段示例（含 `refreshing` ref、`try/finally`、`:disabled`、数组那行注释）；推荐用实例方法那条 note。
- `api/utils/clear-nuxt-data`：定位句 "Delete cached data, error status and pending promises of useAsyncData and useFetch."；用途 note（"invalidate the data fetching for another page"）；签名（含谓词类型与返回 `void`）；Parameters 句（不给键时 all data invalidated）。
- `api/composables/use-async-data`、`api/composables/use-fetch`：实例方法 `refresh`/`execute`/`clear` 的说明与"上次没跑完会被等住"那句（214 已逐句引）。
- `directory-structure/app/pages`、`api/utils/navigate-to`：keepalive 与跳转的用法出处（137/124 已逐句引）。
