---
url: /posts/4dd9aa40b607fd53f5ac5378ef17dcbb/
title: 返回值七个成员：data、status、pending、error、refresh、execute、clear
date: 2026-12-22T10:00:00+08:00
updated: 2026-12-22T10:00:00+08:00
author: cmdragon

summary:
  摘要：`useAsyncData` 返回的是"一个对象同时又是 Promise"，官方 Return Values 一节的原话是："This composable returns a `Promise` that can be awaited, which makes it possible to use `data` directly within the `<script setup>` (i.e. a value will be present, instead of being undefined). You can also directly pull the values without awaiting the return value, in which case `data` can be undefined within `<script setup>` until the fetch completes."。表格里的**七个成员**逐个抄：`data`（`Ref<DataT \| undefined>`，"The result of the asynchronous function that is passed in."）、`refresh`（手动刷新，且"By default, Nuxt waits until a `refresh` is finished before it can be executed again."）、`execute`（官方只给两个字："Alias for `refresh`."）、`error`、`status`（四个取值 `idle`/`pending`/`success`/`error`，并各有一句定义）、`pending`（`true` while a request is in flight，还有一条 `experimental.pendingWhenIdle` 的附加行为）、`clear`（一句里干了四件事：把 `data` 复位成 `undefined` 或 `options.default()`、清 `error`、把 `status` 置 `idle`、取消进行中的调用）。类型层面还有两条容易被跳过：note 说 `data`/`status`/`pending`/`error` 是 ref、`refresh`/`execute` 与 `clear` 是普通函数；tip 说没 await 时 `then`/`catch`/`finally` 也可以安全解构。**另有一处官方示例自身不严谨，我在文末单独标了出来。**

categories:
  - 前端开发

tags:
  - nuxt4
  - useAsyncData
  - status
  - refresh
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


212 里那个解构写了六个成员（`data, status, pending, error, refresh, clear`），少了 `execute`。这一篇把七个补齐，并把"哪些是 ref、哪些是函数"这件事说清——它直接决定你在 `<script setup>` 里要不要加 `.value`。

## 七个成员（逐行照抄）

| 成员 | 类型 | 官方说明 |
| --- | --- | --- |
| `data` | `Ref<DataT \| undefined>` | "The result of the asynchronous function that is passed in." |
| `refresh` | `(opts?: AsyncDataExecuteOptions) => Promise<void>` | "Function to manually refresh the data. By default, Nuxt waits until a `refresh` is finished before it can be executed again." |
| `execute` | `(opts?: AsyncDataExecuteOptions) => Promise<void>` | "Alias for `refresh`." |
| `error` | `Ref<ErrorT \| undefined>` | "Error object if the asynchronous function threw an error." |
| `status` | `Ref<'idle' \| 'pending' \| 'success' \| 'error'>` | "Status of the asynchronous function call. Use it to distinguish `idle`, `pending`, `success`, and `error`." |
| `pending` | `Ref<boolean>` | "`true` while a request is in flight."（外加 `pendingWhenIdle` 那条，见下） |
| `clear` | `() => void` | "Resets `data` to `undefined` (or the value of `options.default()` if provided), `error` to `undefined`, set `status` to `idle`, and cancels any pending calls." |

ref 与函数的分界，官方在同页给了一条 note：

> `data`, `status`, `pending`, and `error` are Vue refs. Access their values with `.value` in `<script setup>`. `refresh`/`execute` and `clear` are plain functions.

**四个 ref、三个函数**。这条在模板里不用管（模板会自动解包 ref），但写 `<script setup>` 逻辑时最容易忘——比如 `if (status === 'pending')` 永远不成立，因为 `status` 是 ref 对象，要写 `status.value`。

## status 四个值，各有触发条件

官方在 Status Values 小节给了四行定义，其中两条直接绑到了选项上：

- `idle`: Function has not been called yet (e.g. `{ immediate: false }` or `{ server: false }` on server render)
- `pending`: Function has been called and the promise is pending
- `success`: Function returned a value
- `error`: Function threw an error

我把这四行和 `pending` 那条说明叠起来读，能得出一件很有用的事：**`pending` 与 `status === 'pending'` 不是同一件事**。`pending` 的说明里有一句例外：

> `true` while a request is in flight. With `experimental.pendingWhenIdle`, it is also `true` when `status` is `idle` and no cached data is available.

也就是说开了 `pendingWhenIdle` 之后，**"还没开始"也会被算成"忙"**——这正是列表页那种"既不显示空状态也不显示内容"的时刻该有的行为。这个开关本身在实验特性那一节，本篇不展开它的取值。

## refresh 与 execute：别名，但参数不是空的

`execute` 官方只写"Alias for `refresh`."，所以两者的差别我**只当作命名差别**来处理，文档没说它们有任何行为不同。

值得注意的是两者的入参类型 `AsyncDataExecuteOptions`：

```ts
interface AsyncDataExecuteOptions {
  dedupe?: 'cancel' | 'defer'
  timeout?: number
  signal?: AbortSignal
}
```

**三个可选键**：这一次执行可以临时改 `dedupe` 策略、临时给一个超时、或者传自己的 `AbortSignal`。212 里那个"手动取消"的写法就是走第三个键：

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

再补一句 `refresh` 的默认行为，它解释了为什么连点不会有意义："By default, Nuxt waits until a `refresh` is finished before it can be executed again." ——**上一次没跑完，再来一次会被等住**。这和选项表里 `dedupe: 'cancel' | 'defer'`（默认 `'cancel'`）配合起来才是完整的合并策略，两者关系文档没写成规则，我不硬推。

## clear 一次做四件事

把那句原文拆成清单，它比"清空数据"要重：

1. `data` 复位成 `undefined`（如果提供了 `options.default()`，就用它的值）。
2. `error` 清成 `undefined`。
3. `status` 置为 `idle`。
4. 取消所有进行中的调用。

第 4 条和 212 讲的 signal abort 时机对得上（"The `clear` function is called" 是三种之一）。所以我把它当成**"退出这个数据块"的动作**而不是"清空显示"：调用之后 `status` 是 `idle`，而不是自动重新发起。

## 不 await 时返回值仍然是对象

Return Values 那段（本篇开头引过）加上两条配套提示，构成"到底能不能不 await"的完整答案：

> Even if you do not await the return value, during SSR Nuxt will wait for the request to finish and send the resolved data to the client.

> Functions from the `Promise` (`then`, `catch`, and `finally`) can safely be destructured, if you did not await the return value.

第二条很实用：不 await 时你**可以同时解构 `data` 和 `then`**，因为返回类型写的是 `AsyncData<DataT, DataE> & Promise<AsyncData<DataT, DataE>>`（212 已解释这个交叉类型）。

还有一条 note 是排错时最容易撞上的：

> If you have not fetched data on the server (for example, with `server: false`), then the data *will not* be fetched until hydration completes. This means even if you await `useAsyncData` on the client side, `data` will remain `undefined` within `<script setup>`.

症状非常具体：**你 `await` 了，`data` 还是 `undefined`**，因为服务端那一趟没取、而客户端的取要到水合完成才开始，`await` 并不能提前拿到它。217 专门讲这一档。

## 顺手记一处示例不严谨的地方

同页 Usage 示例解构的是六个成员，但往下几行直接写了 `refresh()` 与 `clear()`，而这两个名字**没有出现在那行解构里**：

```ts
const { data, error } = await useAsyncData('users', (_nuxtApp, { signal }) => $fetch('/api/users', { signal }))

refresh() // will actually cancel the $fetch request (if dedupe: cancel)
clear() // will cancel the latest pending handler
```

这段是文档为了解释 signal 而写的示意，**不能当可运行代码照抄**。我照抄过一回，报错是 `refresh is not defined`。正解很简单：把它加进解构。我记下这条只有一个目的——看到官方示例里的裸调用时，先确认它有没有对应的解构。

## 自查

1. 七个成员里哪四个是 ref、哪三个是函数？（答：`data`/`status`/`pending`/`error` 是 ref；`refresh`/`execute`/`clear` 是函数。）
2. `refresh` 与 `execute` 的差别文档说了什么？（答：只说了 "Alias for `refresh`."，没给任何行为差异。）
3. 明明 `await` 了，`data` 为什么还是 `undefined`？（答：`server: false` 时数据要等水合完成才取，官方那条 note 讲的就是这个。）

## 下一篇

215 - key 从哪来：自动生成、共享状态与那七个必须一致的选项

## 本文依据

- `api/composables/use-async-data`：Return Values 段全文；两条 tip（SSR 会等完再发、`then`/`catch`/`finally` 可解构）；`server: false` 那条 note；7 行返回值表全部原文；`AsyncData` 类型定义（七个成员的类型）；`AsyncDataExecuteOptions` 三键；`AsyncDataRequestStatus` 四值与 Status Values 四条；note（四个 ref 与三个函数）；`pendingWhenIdle` 那句附加说明；Make Your `handler` Abortable 一节里手动 abort 示例与那段裸调用 `refresh()`/`clear()` 的示例形状。
- 与 217 的分工：`server: false` 的完整后果（首屏无数据、水合后才取）在 217；本篇只把它作为"await 也拿不到值"的症状点出来。
