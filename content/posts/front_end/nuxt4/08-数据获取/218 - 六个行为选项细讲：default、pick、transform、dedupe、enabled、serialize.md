---
url: /posts/7a7037124074770d4f3d2f2057457d34/
title: 六个行为选项细讲：default、pick、transform、dedupe、enabled、serialize
date: 2026-12-23T11:00:00+08:00
updated: 2026-12-23T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：213 那张表把 14 个键列全了，这一篇挑六个"会改变数据本身或改变请求节奏"的讲透，并且只讲文档写了的部分。`default` 是工厂函数（"Factory for default value of `data` before async resolves."），它还是 `clear()` 的复位目标——官方在 `clear` 那一行写的是 "Resets `data` to `undefined` (**or the value of `options.default()` if provided**)"。`pick` 一句到底："Only pick specified keys from the result."，用法是 `{ pick: ['title'] }`。`transform` 是 "Function to transform the result after resolving."，类型允许返回 Promise。**`dedupe` 官方只给了名字和一句话**：`'cancel' | 'defer'`、默认 `'cancel'`、说明是 "Policy when triggering an execution more than once at a time."（`useFetch` 那页换成 "Avoid fetching same key more than once at a time."）；能从别处推出来的只有半句——212 抄过的那条 abort 时机里点名 "A new request is made with `dedupe: 'cancel'`"，所以 `'cancel'` 会取消旧请求，**`'defer'` 具体推迟什么文档没写**。`enabled` 是三道门的总闸，`serialize` 决定要不要进 payload。最后一条属于"清理旧知识"：**4.x 文档里没有 `getPreviousData` 这个选项**，我在讲这一族选项时全文检索过它，零命中；旧文章里靠它做"翻页时先显示上一页"的写法，在 4.x 的替代是 215 那招——用 `useNuxtData` 读缓存 + `default()` 兜默认值。

categories:
  - 前端开发

tags:
  - nuxt4
  - transform
  - pick
  - dedupe
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


这六个键的共同点是：**它们不影响"在哪一侧跑"，影响的是数据长什么样、以及请求什么时候算数。** 所以我把它们和 217 那一档分开讲。

## default：既是开场值，也是复位目标

定义只有一句："Factory for default value of `data` before async resolves."，类型 `() => DataT`。

它出现在两个地方，第二个我一开始没往这上面想：

- 请求还没回来时 `data` 的值（所以骨架屏可以直接省掉——`default` 给的就是可渲染的内容）。
- `clear()` 的复位目标。官方那行原文：「Resets `data` to `undefined` (or the value of `options.default()` if provided), `error` to `undefined`, set `status` to `idle`, and cancels any pending calls.」

**给了 `default` 之后 `clear()` 不会把 `data` 变成 `undefined`**，这一点决定了"清空"之后界面是空状态还是占位内容。

215 那个例子是 `default` 最有用的形态——先拿别的缓存顶上：

```ts
default () {
  // Find the individual post from the cache and set it as the default value.
  return posts.value.find(post => post.id === route.params.id)
}
```

注意它是**工厂**（每次要默认值时调用），不是一个写死的对象。

## pick 与 transform：都在"结果之后"，顺序文档没说

两条定义：

- `pick`（`string[]`）："Only pick specified keys from the result."
- `transform`（`(input: ResT) => DataT | Promise<DataT>`）："Function to transform the result after resolving."

`pick` 的用法示例（`useFetch` 页 Usage）：

```vue [app/pages/modules.vue]
<script setup lang="ts">
const { data, status, error, refresh, clear } = await useFetch('/api/modules', {
  pick: ['title'],
})
</script>
```

这里有个我必须先说明白的空白：**两条都写着"在结果之后"，但谁先跑，文档没有写。** 我从类型上能看到的只有 `useAsyncData` 签名里那句 `AsyncDataOptions<ResT, DataT = ResT>`——`DataT` 默认等于 `ResT`，也就是"不 transform 时最终类型就是 handler 的返回类型"。所以我不会画"handler → transform → pick"这种流水线图。要知道顺序，办法只有一个：两个都用上，`pick: ['title']` 配一个把对象改形的 `transform`，看最终 `data` 里剩什么——**顺序不同结果明显不同，一次就能测出来**。

另外 `pick` 的语义是"只保留指定键"，而 `transform` 可以做任意映射；两者在 215 那张"必须一致"的清单里都在，同 key 的调用不能一个用 `pick` 一个不用。

## dedupe：官方只给名字，不给两种取值的解释

原文两条：

- `useAsyncData`："Policy when triggering an execution more than once at a time."，`'cancel' | 'defer'`，默认 `'cancel'`。
- `useFetch`："Avoid fetching same key more than once at a time."（同一件事的另一种说法）。

**这两个取值的差别文档没有展开。** 能从别处拼出来的只有 `'cancel'` 那半边：212 抄过的 abort 时机第一条是 "A new request is made with `dedupe: 'cancel'`"——所以 `'cancel'` 下新请求会取消旧请求。至于 `'defer'` 是排队等旧的跑完、还是直接复用旧结果，文档一个字没写，我不补。

还有一句容易忽略的默认行为在 `refresh` 那一行（214 抄过）：

> Function to manually refresh the data. By default, Nuxt waits until a `refresh` is finished before it can be executed again.

所以"连点刷新"这件事本身已经被等待挡住了，`dedupe` 管的是**同一时刻并发**的策略。两者边界文档没写成规则。

另外 `dedupe` 不只出现在选项里，它也是单次调用的参数：`AsyncDataExecuteOptions` 三键里就有 `dedupe?: 'cancel' | 'defer'`（和 `timeout`、`signal` 并列），意思是**这一次执行可以临时改策略**。

## enabled：三种触发一起挡的总闸

整段抄下来，因为它每一句都是行为：

> Barrier that gates whether the `handler` may run. While `false`, every execution is blocked (initial fetch, `execute`/`refresh`, and watch triggers), and switching `true` → `false` cancels any in-flight request without clearing `data`. Re-enabling does not refetch on its own.

四个要点：**挡住首次/手动/watch 三类**、切 `false` 取消在飞的请求、**不清空 `data`**、切回 `true` 不自动重取。

它和 `immediate: false` 的分工是"要不要立刻开跑"对"现在允不允许跑"。这两者都会让 `status` 停在 `idle`（`idle` 的定义点名了 `immediate: false` 与 `server: false`，没点 `enabled`）。

## serialize：进不进 payload

整段（含官方给的两个后果）：

> Whether to store resolved data in the Nuxt payload (`__NUXT_DATA__`). When `false`, server-fetched data is kept out of the payload and the client will refetch after hydration if a component renders it. Pair with lazy hydration to avoid hydration mismatches and unnecessary client fetches.

三个信息：默认 `true`；关掉后**服务端那次仍然执行**（与 `server: false` 的区别见 217）；客户端重取的条件是"有组件在渲染它"，不是无条件的。官方还建议搭配延迟水合——这条建议的含义是：既然数据不带过去，就别让这块内容参与水合，否则水合两侧内容不同，正好落回 199 那张表的第一类成因。

顺带一提，`__NUXT_DATA__` 这个名字在这里出现，200 讲 `_payload.json` 时也碰到过同一族概念（内联还是抽文件由 `experimental.payloadExtraction` 决定）。`serialize: false` 是**按 key 摘掉某一块**，`payloadExtraction` 是**整体搬运方式**，两者不是一回事。

## 顺手清一个旧知识：getPreviousData

我在核对这一族选项时把 `getPreviousData` 全文检索过：**4.x 文档里不存在这个选项**，零命中。旧版教程里它常见的写法是"上一次的结果先留着，翻页时不至于白屏"。

在 4.x 的文档里，同一个诉求有两条被明写出来的替代路径，都是前面讲过的：

1. `useNuxtData(key)` 读缓存 + `default()` 兜默认值（215 那个 posts/post-id 例子）。
2. `getCachedData` 自己决定这次用哪个来源——它的默认实现就是按"水合中读 `payload.data`、否则读 `static.data`"来选的（213），第三个参数还给了 `ctx.cause` 四个来源值（216）。

这两条都不是我为了填补空白硬凑的，是官方各自有原文的写法。

## 自查

1. 给了 `default` 之后调用 `clear()`，`data` 是什么？（答：不是 `undefined`，而是 `options.default()` 的值。）
2. `dedupe` 两个取值的差别文档写了吗？（答：没写；只有 `'cancel'` 会取消在飞的请求这一条能从 abort 时机推出。）
3. 想在 4.x 里做"先显示上一次数据"，文档给出的两条路是什么？（答：`useNuxtData` + `default()`；自定义 `getCachedData`。）

## 下一篇

219 - useFetch 与 useAsyncData 的六处差异：自动 key、可响应式 URL 与错误类型

## 本文依据

- `api/composables/use-async-data`：选项表 `default`/`transform`/`pick`/`deep`/`dedupe`/`enabled`/`serialize`/`middleware`/`getCachedData` 各行原文与默认值、版本徽标；返回值表里 `clear` 与 `refresh` 两行原文；`AsyncDataOptions` 与 `AsyncData<DataT, DataT>` 类型（含 `DataT = ResT`）；`AsyncDataExecuteOptions` 三键；signal abort 三条时机；getCachedData 默认实现与 `payloadExtraction` 限定句；Shared State 两张清单（215 已逐条引）。
- `api/composables/use-fetch`：Usage 里 `{ pick: ['title'] }` 示例；`dedupe` 行 "Avoid fetching same key more than once at a time."；`serialize`、`enabled`、`default`、`transform`、`getCachedData` 各行原文；同一条 `clear`/`refresh` 返回值描述；Status Values 四条。
- `api/composables/use-nuxt-data`：`default()` 读缓存的完整示例（215 已引）。
- 检索结论（否定性事实）：`getPreviousData` 在 4.x 文档中零命中。
