---
url: /posts/eb3d59a7c7cb37ca1f712a80d7b075d4/
title: 253 - clearNuxtState 按 key 清理状态与那个 reset 选项
date: 2027-01-04T10:00:00+08:00
updated: 2027-01-04T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`clearNuxtState` 是 `useState` 那一侧的清理开关，整页很短，我把全部四句原文摆出来。**定位**："Delete the cached state of useState."；**用途**："This method is useful if you want to invalidate the state of `useState`. You can also reset the state to its initial value by passing `{ reset: true }` as the second parameter."；**签名** `clearNuxtState (keys?: string | string[] | ((key: string) => boolean), opts?: ClearNuxtStateOptions): void`——**键的第三种形态是谓词函数**，而"不给键时 **all state** will be invalidated"。最有后果的是第二个参数的默认值：官方写 "When not specified, defaults to the value of `experimental.defaults.useState.resetOnClear` in your Nuxt config (**which is `true` with `compatibilityVersion: 5`**)."——**"清空"到底变 `undefined` 还是回到 `init` 的初值，取决于兼容版本**。这一篇把它与取数侧的 `clearNuxtData`、实例的 `clear()` 对齐（229/214 记过措辞差异），并说明 `experimental.defaults.*` 这个命名空间在 4.x 至少出现在两处（另一处是 `<NuxtLink>` 的 `prefetchOn`），所以它是配置层的通用形态，不是给状态开的特例。

categories:
  - 前端开发

tags:
  - nuxt4
  - clearNuxtState
  - useState
  - 清理
  - 状态

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先解释"清空状态"为什么有两种含义：一个计数器清零，你要的是它显示 `0`（初值），还是显示空（`undefined`）？两种在产品里都常见，而**默认走哪一条不由你决定**。

## 整页四条原文

| 项目 | 官方原话 |
| --- | --- |
| 定位 | "Delete the cached state of `useState`." |
| 用途 | "This method is useful if you want to invalidate the state of `useState`. You can also reset the state to its initial value by passing `{ reset: true }` as the second parameter." |
| `keys` | "One or an array of keys that are used in `useState` to delete their cached state. If no keys are provided, **all state** will be invalidated." |
| `opts.reset` | "When set to `true`, resets the state to the initial value provided by the `init` function of `useState` instead of setting it to `undefined`. When not specified, defaults to the value of `experimental.defaults.useState.resetOnClear` in your Nuxt config (which is `true` with `compatibilityVersion: 5`)." |

签名：

```ts
export function clearNuxtState (keys?: string | string[] | ((key: string) => boolean), opts?: ClearNuxtStateOptions): void
```

三件事一次说清：**三种键形态**（字符串、数组、谓词）、**返回 `void`（不可 await）**、**不给键就是全清**。

## 谓词这一档：能按前缀批量清

`keys` 的类型里那个 `(key: string) => boolean` 是这页最有用的能力，因为 `useState` 的 key 散在全应用：

```ts
// 我的写法（类型允许，文档没有这种示例）
clearNuxtState(key => key.startsWith('user:'))
```

**这一行必须标清楚**：官方参数说明只写了"一个字符串或字符串数组"，谓词只出现在**签名的类型**里，整页没有示例。所以我把它当"类型许可、行为未示范"，用之前测一次：设两格 `user:profile` 与 `cart`，调用后各自读回，看被清的是不是只有前者。

229 里 `clearNuxtData` 是同一形状（签名里有谓词、无示例），两处我按同一标准处理。

## reset 的默认值会随兼容版本变

这条最该记住：

> When not specified, defaults to the value of `experimental.defaults.useState.resetOnClear` in your Nuxt config (which is `true` with `compatibilityVersion: 5`).

拆开看四种情形：

| 情形 | 清空后 `useState` 的值 |
| --- | --- |
| 显式 `{ reset: true }` | 回到 `init()` 的初值 |
| 显式 `{ reset: false }` | `undefined` |
| 不指定、兼容版本非 5 | 跟随 `experimental.defaults.useState.resetOnClear` |
| 不指定、`compatibilityVersion: 5` | 回到初值 |

**升级时"清空"的含义可能翻转**。所以可执行结论只有一条：在意结果就把 `opts` 写全（`clearNuxtState('counter', { reset: true })`），别把行为寄托在版本默认值上。

顺带一句推论：`init` 是工厂（官方说明是 "provides initial value **when not initiated**"），所以 reset 回的是**再调一次 `init()`** 的结果——如果 `init` 里是 `Math.random()`，reset 后得到的是**新随机值**而不是原值。这句是两处原文相加的结果，官方没写在一起，用前测一次。

## 与取数侧三个动作对齐

| 动作 | 管的对象 | 复位到哪 | 返回 |
| --- | --- | --- | --- |
| `clearNuxtState(keys?, opts?)` | `useState` 的状态 | `undefined` 或 `init()` 初值（看 `reset`） | `void` |
| `clearNuxtData(keys?)` | "cached data, error status and pending promises of useAsyncData and useFetch" | 官方只说 "delete their cached data" | `void` |
| 实例的 `clear()` | 那一格 | "`data` to `undefined` (or the value of `options.default()` if provided)"，`status` 置 `idle`、取消在飞调用 | `void` |
| `refreshNuxtData(keys?)` | 四个取数 composable（含 `useLazy*`） | 不是清，是重取 | 可 await 的 Promise |

**状态与取数是两套 key 空间**（`clearNuxtState` 只点 `useState`，`clearNuxtData` 只点 `useAsyncData`/`useFetch`），所以"退出登录"通常两边都要清：

```ts
clearNuxtState(key => key.startsWith('user:'))
clearNuxtData(['profile', 'unread-count'])
```

（谓词那行仍是"类型允许、无示例"。）

## 这个 defaults 命名空间不是特例

`experimental.defaults.*` 至少两处出现，形态一致——**为某个 composable 的选项设全局默认值**：

| 默认值位置 | 管的选项 |
| --- | --- |
| `experimental.defaults.useState.resetOnClear` | `clearNuxtState` 的 `reset`（本页） |
| `experimental.defaults.nuxtLink.prefetchOn` | `<NuxtLink>` 何时预取（性能页的例子是改成 `interaction: true, visibility: false`） |

**这解释了一类"我明明配了却没生效"的排错方向**：这类开关改的是**默认值**，任何一次显式传参都会盖掉它。读到"`defaults` 里有我的选项"时，我第一步是去调用点搜有没有硬编码值，而不是怀疑开关无效。

## 我的清理写法

1. **能列 key 就列 key**：`clearNuxtState(['user:profile', 'cart'])`，避免全清把无关状态一起打回。
2. **在意"清空后显示什么"就写死 `reset`**，不依赖默认。
3. **状态与取数分开清**，两边各一次；只有 `refreshNuxtData` 能 await，所以"清完再跳"要自己排顺序。
4. 组件在 `<KeepAlive>` 里时（246 说它被隐藏仍会被重取），清状态后模板可能立刻读到新值——**清理不是卸载**。

## 自查

1. `clearNuxtState()` 不给参数会怎样？（答：全部状态失效——"all state will be invalidated"；返回 `void`。）
2. `reset` 不指定时由什么决定？（答：`experimental.defaults.useState.resetOnClear`，官方注明 `compatibilityVersion: 5` 下为 `true`。）
3. 为什么退出登录通常要调两个函数？（答：`clearNuxtState` 只管 `useState`，取数那一格要 `clearNuxtData` 或实例的 `clear()`。）

## 下一篇

254 - 状态与取数怎么搭配：default 读缓存与 transform 重塑
