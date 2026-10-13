---
url: /posts/e42b7ddab6016469e4a18a2d32629e30/
title: 230 - callOnce：只跑一次那一档与 navigation 模式
date: 2026-12-27T11:00:00+08:00
updated: 2026-12-27T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：212 那条 warning 说"handler 要无副作用，需要副作用就用 `callOnce`"，这一篇就是那把工具的说明书。官方定位是 "Run a given function or block of code once during SSR or CSR."（v3.9 起可用），Purpose 一节把"一次"定义成两个场合："execute a given function or block of code only once during: **server-side rendering but not hydration** / **client-side navigation**"，并给出适用句："This is useful for code that should be executed only once, such as logging an event or setting up a global state."。默认模式的解释很关键："The default mode of `callOnce` is to run code only once. For example, if the code runs on the server it won't run again on the client. **It also won't run again if you `callOnce` more than once on the client, for example by navigating back to this page.**"；想要"每次客户端导航都跑一次"就显式开 `navigation` 模式（v3.15 起），参数说明写得很对称：`render` 是 "Executes once during initial render (either SSR or CSR) - Default mode"，`navigation` 是 "Executes once during initial render and once per subsequent client-side navigation"。**但它不是取数工具**——warning 原话："Note that `callOnce` doesn't return anything. You should use `useAsyncData` or `useFetch` if you want to do data fetching during SSR."，而它之所以能跨水合去重，是因为 note 里那句"需要往 Nuxt payload 里加数据"。

categories:
  - 前端开发

tags:
  - nuxt4
  - callOnce
  - 副作用
  - SSR
  - Pinia

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先解释我为什么需要它：同一段代码在服务端跑一次、浏览器水合再跑一次，是 SSR 的正常行为（210）。对取数这是浪费但有解；**对"上报一次埋点"或"初始化一次全局状态"，跑两次就是 bug**——`callOnce` 管的是后者。

## 它保证的是哪两个"一次"

Purpose 一节原文：

> The `callOnce` function is designed to execute a given function or block of code only once during:
> - server-side rendering but not hydration
> - client-side navigation

第二行那句适用范围（我觉得容易被略过）："This is useful for code that should be executed only once, such as logging an event or setting up a global state."

默认模式的行为，官方给了两句我认为最实用的描述：

> The default mode of `callOnce` is to run code only once. For example, if the code runs on the server it won't run again on the client. It also won't run again if you `callOnce` more than once on the client, for example by navigating back to this page.

拆开是两个保证：**服务端跑过了，水合就不再跑**；**同一份代码在客户端反复触发（比如来回导航）也只跑一次**。第二个保证是靠 key 做到的——参数说明写得很清楚："A unique key ensuring that the code is run once. If you do not provide a key, then a key that is unique to the file and line number of the instance of `callOnce` will be generated for you."（跟 215 讲的自动 key 同一套规则，**所以封装函数里同样要注意调用位置只有一个**。）

## 官方示例：配 useState 是最常见的形状

```vue [app/app.vue]
<script setup lang="ts">
const websiteConfig = useState('config')

await callOnce(async () => {
  console.log('This will only be logged once')
  websiteConfig.value = await $fetch('https://my-cms.com/api/website-config')
})
</script>
```

这段的分工值得看清楚：`callOnce` 保证**只跑一次**，`useState` 保证**跑完的值能跨过水合存在**（100 讲过 `useState` 是 SSR 友好的全局响应式状态）。**如果这里没有 `useState` 而是局部 `ref`，那么水合后新起的 ref 就是空的**——这一层是 `callOnce` 不管的。

## navigation 模式：另一种"一次"

> It is also possible to run on every navigation while still avoiding the initial server/client double execution. For this, it is possible to use the `navigation` mode :v3.15

```vue [app/app.vue]
<script setup lang="ts">
const websiteConfig = useState('config')

await callOnce(async () => {
  console.log('This will only be logged once and then on every client side navigation')
  websiteConfig.value = await $fetch('https://my-cms.com/api/website-config')
}, { mode: 'navigation' })
</script>
```

参数说明那两行把两者对齐得最干净：

- `render`: Executes once during initial render (either SSR or CSR) - Default mode
- `navigation`: Executes once during initial render and once per subsequent client-side navigation

类型定义里默认值也写明了：`CallOnceOptions = { mode?: 'navigation' | 'render' }`，注释 `@default 'render'`。

**`navigation` 模式解决的是"每次进这个页面都重新拿一次，但不要两侧各拿一次"**——它和 216 讲的响应式重取是两码事（那是数据跟着 ref 变，这是跟着导航变），也和 224 的 `lazy` 无关（那是阻不阻塞导航）。我原来把这三者混成一团，把参数表分开读才理清。

## 三条限制比用法更重要

第一条是这个工具最容易被误用的地方，warning 原文：

> Note that `callOnce` doesn't return anything. You should use `useAsyncData` or `useFetch` if you want to do data fetching during SSR.

**返回 void，所以没有 `data`、没有 `status`、没有 `error`。** 上面示例里那句 `websiteConfig.value = await $fetch(...)` 是作者自己写进去的，框架不会帮你把结果交出来，也不会帮你处理这次请求的失败。

第二条是调用位置，note 说得比别的 composable 更具体：

> `callOnce` is a composable meant to be called directly in a setup function, plugin, or route middleware, because it needs to add data to the Nuxt payload to avoid re-calling the function on the client when the page hydrates.

后半句我认为值得单独记：**它能去重的机制就是往 payload 里写一条记录**。所以它和 213 那条"缓存只在 `payloadExtraction` 开启时有效"是同一个量级的依赖——payload 那条通道不成立时，去重就不成立。

第三条是版本：两条 important，**整个工具 "available since Nuxt v3.9"**，**`navigation` 模式 "available since Nuxt v3.15"**。

签名两条也照抄，因为 key 是可选项：

```ts
export function callOnce (key?: string, fn?: (() => any | Promise<any>), options?: CallOnceOptions): Promise<void>
export function callOnce (fn?: (() => any | Promise<any>), options?: CallOnceOptions): Promise<void>
```

注意返回类型是 `Promise<void>`——**`await` 它能等到这次执行完成**（跟"不返回任何值"不矛盾：等得到时机，拿不到结果）。

## 官方点名的那个组合：Pinia

tip 一句："`callOnce` is useful in combination with the Pinia module to call store actions."，并指向状态管理那一节。

这正好是 212 那条 warning 的落地形状：store 的 action 是典型副作用，**把它放进 `useAsyncData` 的 handler 里就违反"handler 无副作用"，放进 `callOnce` 里就对了**。

## 什么时候用哪个，我的判据

| 我要做的事 | 用什么 |
| --- | --- |
| 首屏要渲染这份数据 | `useAsyncData` / `useFetch`（210/219） |
| 只想跑一次的初始化、埋点、store action | `callOnce`（默认 `render`） |
| 每次进入页面都要重新初始化，但不要两侧双跑 | `callOnce(..., { mode: 'navigation' })` |
| 数据跟着某个 ref 变 | `watch` 或响应式选项（216） |
| 改了要乐观更新、失败回滚 | `useNuxtData` + 拦截器（228） |

## 自查

1. `callOnce` 默认保证的两次"不跑"分别是什么？（答：服务端跑过则水合不跑；客户端反复触发（如来回导航）也只跑一次。）
2. 为什么它不能用来做首屏取数？（答：它不返回任何值——没有 `data`/`status`/`error`，官方明说取数要用 `useAsyncData` 或 `useFetch`。）
3. `render` 与 `navigation` 两档的差别一句话是什么？（答：前者只在初始渲染跑一次，后者初始一次、之后每次客户端导航再各一次。）

## 下一篇

231 - onPrehydrate：水合前内联进 HTML 的那段脚本

## 本文依据

- `api/utils/call-once`：定位句 "Run a given function or block of code once during SSR or CSR."；v3.9 important；Purpose 两条与适用句；默认模式解释段（含"回导航也不再跑"那句）；两段示例（`useState` + `$fetch`，以及 `{ mode: 'navigation' }`）与 v3.15 important；Pinia 那条 tip 与指向 state-management 的 read-more；warning（doesn't return anything、取数请用 useAsyncData/useFetch）；note（setup/插件/路由中间件、需要往 payload 加数据）；Type 两条重载与 `CallOnceOptions`（`mode?: 'navigation' | 'render'`、`@default 'render'`）；Parameters 三条（key 的文件名+行号自动生成、fn 可异步、options 两档各自的原话）。
- `api/composables/use-async-data`：side-effect free 那条 warning 与"用 `callOnce`"的指路（212 已逐句引）；payload 与 `payloadExtraction` 的关系（213 已逐句引）。
- `api/composables/use-state`：SSR 友好全局状态的推荐句（100 已逐句引）。
