---
url: /posts/f297fbf074edf0fa6a6cfb445b2ac313/
title: useState 的 key 规则与序列化约束
date: 2027-01-04T09:00:00+08:00
updated: 2027-01-04T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把 `useState` 的两条硬规则合起来讲：**key 怎么定、什么能放进去**。key 一侧，官方说明是 "A unique key ensuring that data fetching is properly de-duplicated across requests. If you do not provide a key, then a key that is unique to the file and line number of the instance of `useState` will be generated for you."，两条校验各有错误码：**E7009**（"useState() was called with a key that is not a string. The key identifies the shared state across the app **and in the hydration payload**, so it must be a non-empty string."）与 **E7007**（"useState() received a non-function initial value. The initializer must be a function **so that it only runs once, on the server, and its result can be serialized into the payload for hydration.**"）。**E7007 那句理由我一字一句读**：初值必须是函数，是因为它只在服务端跑一次、结果要能序列化进水合用的 payload——它同时回答了"`useState` 为什么能跨水合"与"为什么放不进类实例"。序列化一侧，官方口径是 "serialized to JSON"，报错是 `Cannot stringify arbitrary non-POJOs`，解法是 `definePayloadPlugin` 注册自定义 reducer/reviver；而 payload 那一页的说法是 `devalue`（能带 Date/Map/Set/ref 等）——**两处口径不一致，我按最严的写并把差异列成实测项**。清理一侧是 `clearNuxtState(keys?, opts?)`：定位句 "Delete the cached state of useState."、参数支持 **谓词函数**、"If no keys are provided, **all state** will be invalidated"，v4.4 起多一个 `reset` 选项（"resets the state to the initial value provided by the `init` function of useState instead of setting it to `undefined`"），不指定时跟随 `experimental.defaults.useState.resetOnClear`（**`compatibilityVersion: 5` 下为 `true`**）。

categories:
  - 前端开发

tags:
  - nuxt4
  - useState
  - key
  - clearNuxtState
  - 序列化

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`useState` 只有两个参数，但两个参数各带一条校验、各有一个会踩的坑。这一篇按参数来讲。

## key：字符串、非空、且决定归属

官方参数说明（照抄）：

> `key`: A unique key ensuring that data fetching is properly de-duplicated across requests. If you do not provide a key, then a key that is unique to the file and line number of the instance of [`useState`] will be generated for you.

E7009 把"为什么这么严"说了：

> `useState()` was called with a key that is not a string. The key identifies the shared state across the app **and in the hydration payload**, so it must be a non-empty string.

**"在 hydration payload 里"这半句是整件事的关键**：key 不只是查找用的名字，它同时是**水合数据里的字段名**。所以我给 key 起名的规则和给字段起名一样——要稳定、要唯一、别在循环里生成一堆随机值。

不给 key 时会怎样？按上面那句，是"文件名 + 行号"。这跟 `useAsyncData` 完全同一套（215 引过），所以同一类坑也照搬过来：

| 情形 | 结果 |
| --- | --- |
| 两个组件各自 `useState('theme', ...)` | 同一格，共享 |
| 同一个 `useState('theme', ...)` 在不同文件里各写一次 | 各自一格（key 相同才共享，这里是 key 相同 ⇒ 其实仍共享） |
| **不写 key** 的包装 composable | 只有生成处的一个位置，所有调用方共用同一格 |

第三行才是那条自动规则真正咬人的地方（官方对 `useAsyncData` 给的建议同样适用：**"it is recommended to always create your own key"**）。第二行我特意留着说明：**给了相同字符串 key 就是同一格，跨文件也一样**——共享单位是 key，不是文件。

## init：必须是函数，理由写在错误码里

E7007 的原文值得整句抄，因为它顺带解释了 `useState` 的机制：

> `useState()` received a non-function initial value. **The initializer must be a function so that it only runs once, on the server, and its result can be serialized into the payload for hydration.**

拆成三条因果：

1. **只在服务端跑一次** ⇒ 初值里的 `Math.random()` 两侧不会各算一次（210 说的"跑两遍"在这里被这条规则挡掉）。这正好解释了官方 Basic Usage 那句 `useState('counter', () => Math.round(Math.random() * 1000))` 为什么可以放心写随机数。
2. **结果要序列化进 payload** ⇒ 它必须可序列化，于是有了下一节那条限制。
3. 参数说明里还有一句 "This function can also return a `Ref`." ⇒ 所以 `shallowRef` 那种包法合法（251）。

`init` 那句限定也别丢："provides initial value **when not initiated**"——**已存在的那格不会被你的 init 覆盖**，这是"多个组件同一 key 各自写 init"时唯一需要小心的点：谁先创建谁定值，后来的 init 不生效。这一条官方没写后果，但按字面就能推，我不再加戏。

## 序列化：两处口径不一致，按最严的来

| 出处 | 说法 |
| --- | --- |
| `useState` 与 state-management 两页的 important | "the data inside `useState` will be **serialized to JSON**... does not contain anything that cannot be serialized, such as classes, functions or symbols" |
| data-fetching 的 payload 一节（235 引过） | payload "is serialized with **devalue**"，能带正则、`Date`、`Map`/`Set`、`ref`/`reactive`/`shallowRef`/`shallowReactive`、`NuxtError` |

两句话的对象其实有重叠（`useState` 的值也要进 payload），**但官方没写清哪个规则适用于哪一类值**。我的处理是：

- **保守线**：类实例、函数、symbol 一律不放（三页都反对）。
- **要放 `Date`/`Map`/`Set` 时先测**：写进 `useState`、`nuxt build` + `nuxt preview`，服务端设值、客户端读回来比对是否为原类型（若按 JSON 走，`Date` 会变成字符串——与 235 那个接口返回 `Date` 的例子同一症状）。
- **确认失败**的报错长这样：`Cannot stringify arbitrary non-POJOs`（POJO 就是"普通 JS 对象"）。官方给的解法不是换容器，而是注册自定义序列化：

> If you want to store class instances with `useState` that are not supported by Nuxt, you can use [`definePayloadPlugin`] to add a custom serializer and deserializer for your classes.

（`definePayloadPlugin` 属于 `useNuxtApp` 的 payload 那一节，与 200 记的 reducer/reviver 是同一条路。）

## 清理：clearNuxtState 的三个参数形状

整页原文很短，四条信息都在：

> Delete the cached state of `useState`.

> This method is useful if you want to invalidate the state of `useState`. You can also reset the state to its initial value by passing `{ reset: true }` as the second parameter.

```ts
export function clearNuxtState (keys?: string | string[] | ((key: string) => boolean), opts?: ClearNuxtStateOptions): void
```

- `keys`："One or an array of keys that are used in `useState` to delete their cached state. If no keys are provided, **all state** will be invalidated."
- `opts.reset`（v4.4）："When set to `true`, resets the state to the initial value provided by the `init` function of `useState` instead of setting it to `undefined`. **When not specified, defaults to the value of `experimental.defaults.useState.resetOnClear` in your Nuxt config (which is `true` with `compatibilityVersion: 5`).**"

**这里有一处与取数侧不对称，我认为值得记住**：`clearNuxtState` 有 `reset` 语义（回到 `init` 的值），而取数侧的 `clear()` 只说"复位到 `default()` 若提供"（214/225 记过措辞差异）。同一个"清空"动作，两族的默认含义不同，**而 `resetOnClear` 的默认值还会随兼容版本变化**——升级 Nuxt 5 时"清空后是空还是初值"这件事可能变。

清理范围我也对齐一遍：`clearNuxtState` 只管 `useState`，`clearNuxtData` 只管 `useAsyncData`/`useFetch`（251 那张表）。**要同时作废两边就各调一次，不给参数就是"全部"。**

## 一页速查

| 我要做的 | 正确写法 | 别这样 |
| --- | --- | --- |
| 跨组件共享一份状态 | 显式同名 key：`useState('theme', () => 'dark')` | 靠自动生成的 key 跨文件共享 |
| 包装成 composable 给别人用 | `export const useColor = () => useState<string>('color', () => 'pink')`（官方原样） | 在模块顶层 `const color = ref('pink')` |
| 存大对象 | `useState('big', () => shallowRef({...}))` | 默认深响应式 + 上万个元素 |
| 存类实例 | 先注册 `definePayloadPlugin` 的 reducer/reviver | 直接塞进去，等 `Cannot stringify arbitrary non-POJOs` |
| 退出登录清状态 | `clearNuxtState(key => key.startsWith('user:'))` 或全清 | 用 `clearNuxtData()` 清状态（管的不是同一个东西） |

（表里谓词那行的写法与 229 一样属于"类型允许但文档没给示例"，用前测一次。）

## 自查

1. 为什么 `init` 必须是函数？（答：官方说它只在服务端跑一次，且结果要能序列化进水合用的 payload。）
2. key 起什么作用？不给会怎样？（答：它同时是应用内共享与水合 payload 里的标识；不给则按文件名+行号自动生成。）
3. `clearNuxtState` 与 `clearNuxtData` 的区别在哪？（答：前者删 `useState` 的状态、可 `reset` 回初值（默认随 `resetOnClear`，cv5 下为 `true`）；后者删 `useAsyncData`/`useFetch` 的数据、错误与在飞请求。）

## 下一篇

253 - clearNuxtState：按 key 精确清理

## 本文依据

- `api/composables/use-state`：两条重载与 `key`/`init`/`T` 三条参数说明（含 "A function that provides initial value for the state when not initiated. This function can also return a `Ref`."）；important 序列化边界；Using `shallowRef` 一节；Troubleshooting 的 `Cannot stringify arbitrary non-POJOs` 与 `definePayloadPlugin`。
- `api/utils/clear-nuxt-state`：定位句；note（"You can also reset the state to its initial value by passing `{ reset: true }"）；签名；`keys` 与 `opts.reset` 两条参数说明（含 v4.4 徽标与 `experimental.defaults.useState.resetOnClear`、`compatibilityVersion: 5` 下为 `true`）。
- `errors/e7007`、`errors/e7009`：两条描述句（初值必须是函数、key 必须非空字符串，及各自关于 payload 的理由）。
- `getting-started/state-management`：Basic Usage 的 `useState('counter', () => Math.round(Math.random() * 1000))` 与共享说明句、Shared State 的 `useColor` 包装、同一条 important（250 已逐句引）。
- `api/composables/use-async-data`：`key` 自动生成句与"建议自己给 key"的同类规则（215 已逐句引）；`clear` 行措辞（214 已逐句引）。
- `api/composables/use-nuxt-data`：覆盖四个取数 composable 且需显式 key（228 已逐句引）；`getting-started/data-fetching` 的 devalue 段与 `api/utils/clear-nuxt-data` 的定位句（235/229 已逐句引）。
