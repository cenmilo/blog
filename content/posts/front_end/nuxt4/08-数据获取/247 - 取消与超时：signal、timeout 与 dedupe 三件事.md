---
url: /posts/b51931582d59f85207527a05e22a6388/
title: 247 - 取消与超时：signal、timeout 与 dedupe 三件事
date: 2027-01-02T10:00:00+08:00
updated: 2027-01-02T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：取数的取消能力全押在一个 `AbortSignal` 上，而官方给的路径一共三条。第一，handler 的第二个参数就是它：`AsyncDataHandler<ResT> = (nuxtApp: NuxtApp, options: { signal: AbortSignal }) => Promise<ResT>`，官方解释是 "You can make your `handler` function abortable by using the `signal` provided in the second argument. This is useful for cancelling requests when they are no longer needed, **such as when a user navigates away from a page**. `$fetch` natively supports abort signals."。第二，**你自己的 signal 可以传进 `refresh`**：`refresh({ signal: abortController.signal })` 配 `abortController?.abort() // aborts the ongoing refresh request`。第三，框架代为取消的时机只有三种（原文逐条）："A new request is made with `dedupe: 'cancel'`"、"The `clear` function is called"、"The `options.timeout` duration is exceeded"。`timeout` 是 v4.2 才有的键，说明句是 "A number in milliseconds to wait before timing out the call (**defaults to `undefined`, which means no timeout**)"，另外两个入参 `dedupe`、`signal` 也在同一个 `AsyncDataExecuteOptions` 里，意味着**这三者都能按单次调用覆盖**。如果你的 handler 不支持 signal，官方给的是自己接 abort 事件（`signal?.addEventListener('abort', () => reject(new Error('Request aborted')))`）。**而"被取消之后 `status` 与 `error` 会变成什么"，文档一句都没写**——这一处我在文末列成必测项。

categories:
  - 前端开发

tags:
  - nuxt4
  - abort
  - timeout
  - dedupe
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先解释一次取消为什么值得单独写一篇：**它决定了用户看到什么**。搜索框里连打五个字，如果不取消，屏幕上出现的是第一个查询的结果——因为最后返回的往往是最早发出的那个请求。

## 三条取消路径

### 路径一：把 handler 收到的 signal 传下去

```ts
const { data, error } = await useAsyncData(
  'users',
  (_nuxtApp, { signal }) => $fetch('/api/users', { signal }),
)
```

官方的说法里两点要抓：

> You can make your `handler` function abortable by using the `signal` provided in the second argument. This is useful for cancelling requests when they are no longer needed, such as when a user navigates away from a page. `$fetch` natively supports abort signals.

**"signal 只是交给你，框架不替你用它"**——第二个参数不传给任何请求，就等于没启用取消。这也是为什么并行请求要**每一个**都带 `{ signal }`（243 那个 `Promise.all` 示例正是这样写的）。

### 路径二：给自己的 refresh 配一个 controller

```ts
let abortController: AbortController | undefined

function handleUserAction () {
  abortController = new AbortController()
  refresh({ signal: abortController.signal })
}

function handleCancel () {
  abortController?.abort() // aborts the ongoing refresh request
}
```

注释是官方的："aborts the ongoing refresh request"。这条路径解决的是"用户点了取消按钮"这类显式意图——框架自己不知道什么时候该停。

### 路径三：框架代劳的三种时机

原文三条：

> The handler signal will be aborted when:
>
> - A new request is made with `dedupe: 'cancel'`
> - The `clear` function is called
> - The `options.timeout` duration is exceeded

对应到调用上就是三个动作：`dedupe` 保持默认 `'cancel'` 时**新请求挤掉旧请求**；调 `clear()`；超过 `timeout`。

同页 Usage 那段还给了这条路径的直观注释：

```ts
refresh() // will actually cancel the $fetch request (if dedupe: cancel)
refresh() // will actually cancel the $fetch request (if dedupe: cancel)
refresh()

clear() // will cancel the latest pending handler
```

（顺带提醒：那段示例本身没解构 `refresh`/`clear`，别照抄，见 214。）

## handler 不支持 signal 时怎么办

官方明确给了手动方案：

> If your `handler` function does not support abort signals, you can implement your own abort logic using the `signal` provided.

```ts
const { data, error } = await useAsyncData(
  'users',
  (_nuxtApp, { signal }) => {
    return new Promise((resolve, reject) => {
      signal?.addEventListener('abort', () => {
        reject(new Error('Request aborted'))
      })
      return Promise.resolve(callback.call(this, yourHandler)).then(resolve, reject)
    })
  },
)
```

读法是：**监听 abort 事件、在事件里 reject 自己那个 Promise**。这意味着取消对第三方 SDK（没有 signal 参数的）也做得到，但要你亲手包。

## timeout：默认是"不超时"

| 项目 | 官方内容 |
| --- | --- |
| 类型 | `number`（毫秒） |
| 默认 | 表里写 `-`，说明句补 "defaults to `undefined`, which means **no timeout**" |
| 版本 | 徽标 v4.2 |
| 说明 | "A number in milliseconds to wait before timing out the call" |

`useFetch` 那页把 "the call" 换成 "the request"，其余一致。

还有一件事容易漏：**`timeout` 也能按单次调用给**——`AsyncDataExecuteOptions` 里就有它：

```ts
interface AsyncDataExecuteOptions {
  dedupe?: 'cancel' | 'defer'
  timeout?: number
  signal?: AbortSignal
}
```

也就是说 `refresh({ timeout: 500 })` 合法。**三个键里 `dedupe` 与 `timeout` 都能在这次调用时临时改**，这是我把它放在 `refresh` 参数而不是全局选项里读的理由。

至于超时之后发生什么，文档只写了"signal 会被 abort"这一半；**`status` 会变成 `'error'` 还是回到 `'idle'`、`error.value` 里放什么，都没写**（见下一节）。

## dedupe 的两档只有一档被写清

`dedupe: 'cancel' | 'defer'`，默认 `'cancel'`，官方说明只有 "Policy when triggering an execution more than once at a time."（`useFetch` 那页换成 "Avoid fetching same key more than once at a time."）。

能确定的只有 `'cancel'`：abort 时机第一条就是它。至于 `'defer'` 是"等旧的跑完再发"还是"这次直接复用旧请求的结果"，**这一族三处记载都没解释**（218 也记过这条空白）。所以我不用 `'defer'`，除非先测出来。

## 三处未说明，我给你测法

| 我不知道的 | 怎么测 |
| --- | --- |
| 被 abort 之后 `status` 是什么 | 起一个慢接口，`refresh()` 后 200 毫秒调 `clear()`，观察 `status.value` 与 `pending.value` 的落点 |
| abort 会不会把 `error.value` 填成一个错误对象 | 同一次实验里打印 `error.value`；若为 `undefined`，模板就得靠 `status` 判断而不是判错误 |
| `timeout` 到点时是不是真的发出 abort | handler 里 `signal.addEventListener('abort', ...)` 打一行日志，配 `{ timeout: 100 }` 打慢接口，看有没有那行 |

这三条都属于"框架行为可测、文档没写"的一类。**测出来的结果建议写进项目 README**，因为它们决定了模板里该判 `status === 'error'` 还是判 `error.value`。

顺带一条能立刻用的：`pending` 那个 ref 在开了 `experimental.pendingWhenIdle` 之后，**`status` 为 `idle` 且没有缓存数据时也会是 `true`**（214 引过）——取消后如果状态回落到 `idle`，这个开关能让界面继续显示加载态而不是空白。

## 自查

1. 三条取消路径分别是什么？（答：把 handler 的 `signal` 传给请求；`refresh({ signal })` 配自己的 controller；框架在 `dedupe:'cancel'` 新请求、`clear()`、超过 `timeout` 三种时机 abort。）
2. `timeout` 的默认值意味着什么？（答：默认 `undefined`，即不超时。）
3. 文档没写、需要自测的是哪三点？（答：abort 后的 `status`/`pending`、`error.value` 是否被填、`timeout` 到点是否真发出 abort。）

## 下一篇

248 - 取数小结：一张决策图表该用哪个 API

## 本文依据

- `api/composables/use-async-data`：Make Your `handler` Abortable 全节（定义句与三块示例、`signal?.addEventListener('abort', ...)` 方案、abort 三条时机）；`AsyncDataHandler` 与 `AsyncDataExecuteOptions` 定义；选项表 `timeout` 行（v4.2 徽标、"defaults to `undefined`, which means no timeout"）、`dedupe` 行与 `pending` 行；Usage 示例里裸调 `refresh()`/`clear()` 的注释形状；`experimental.pendingWhenIdle` 一句（212/213/214/218 已逐句引）。
- `api/composables/use-fetch`：`timeout` 行（"the request"措辞）、`dedupe` 行（"Avoid fetching same key more than once at a time."）、`pending` 行同文（220 已逐句引）。
- `getting-started/data-fetching`：并行请求配方中每个 `$fetch` 都带 `{ signal }` 的写法（243 已逐句引）。
- 与前文的分工：`clear()` 的四件事与状态复位在 214/229，`dedupe` 空白在 218，刷新与导航的并发在 246。
