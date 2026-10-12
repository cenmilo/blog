---
url: /posts/5b5324c0014c4f9c210a516f6740dfc4/
title: 构建期读写运行时配置：useRuntimeConfig 无参与 updateRuntimeConfig 的 HMR
date: 2027-05-17T11:00:00+08:00
updated: 2027-05-17T11:00:00+08:00
author: cmdragon

summary:
  摘要：Kit 这页只有两个函数，**名字与运行时的组合式函数一模一样，含义完全不同**。`useRuntimeConfig` 在这里是**构建期**读取已解析的 runtime config，**签名不接受任何参数**、返回 `Record<string, unknown>`；而运行时那个要么传 event（h3 v1）、要么不传（`nuxt/server`），两者的类型与可见范围都不一样。`updateRuntimeConfig` 是唯一一个会**触发副作用**的：它把传入的配置与已有的合并，并且**如果 Nitro 已经初始化，会发出一个 HMR 事件去重载 Nitro 的 runtime config**——返回类型写成 `void | Promise<void>`，但**页面没有说明什么时候是 `void`、什么时候是 promise**。这一页**连一个示例都没有**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 运行时配置
  - NuxtKit
  - HMR
  - 模块作者
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块要在构建期知道某个配置值，或者直接改运行时配置让宿主读到——这一页给的两个函数就够了。难点不在这页本身，而在于**这两个名字在运行时那侧也存在**。

## 两个函数的全部内容

页面定性：**Nuxt Kit 提供一组工具，帮你访问与修改 Nuxt 的运行时配置**。

**`useRuntimeConfig`**：

> At build-time, it is possible to access the resolved Nuxt runtime config.

```ts
function useRuntimeConfig (): Record<string, unknown>
```

**`updateRuntimeConfig`**：

> It is also possible to update runtime configuration. This will be merged with the existing runtime configuration, and if Nitro has already been initialized it will trigger an HMR event to reload the Nitro runtime config.

```ts
function updateRuntimeConfig (config: Record<string, unknown>): void | Promise<void>
```

**整页到此为止——两个类型块，没有任何示例小节**。

## 先避开同名陷阱

Kit 的 `useRuntimeConfig` 与运行时的 `useRuntimeConfig` 是两套东西。差别可以从三处文档拼出来：

| 位置 | 签名 | 什么时候用 |
| --- | --- | --- |
| **Kit（本页）** | `useRuntimeConfig(): Record<string, unknown>`，**不接受参数** | 构建期，模块代码里 |
| 服务端运行时，h3 v1（Nuxt 4） | 需要把 event 传进去：`useRuntimeConfig(event)` | 服务端处理器里 |
| 服务端运行时，`nuxt/server`（Nuxt 4.6+／Nuxt 5） | 没有按请求的形式，**`useRuntimeConfig()` 不传参** | 同上，可移植写法 |

升级页那张对照表里那一行写的正是"**`useRuntimeConfig(event)` 变成没有按请求形式的 `useRuntimeConfig()`**"。

**这个同名造成的错误很难看出来**：在模块的 `setup` 里从 `@nuxt/kit` 导入它，拿到的是一份 `Record<string, unknown>`，**没有 `public` 与私有两侧的自动分离提示**；而把 Kit 版本误放进服务端文件，就撞上"Kit 只给模块用"那条限制（**发布模块的 runtime 里连自动导入都不启用，必须显式导入**）。

## 改运行时配置有两条路

| 做法 | 出处 | 特点 |
| --- | --- | --- |
| 直接写 `nuxt.options.runtimeConfig.public.myModule`，用 defu 与用户已有的值合并 | 模块基础配方页那一节 | 常规做法，示例明确用 defu 是为了**扩展而不是覆盖**用户提供的公共运行时配置 |
| 调 `updateRuntimeConfig(config)` | 本页 | **合并语义**由函数承担；**Nitro 已初始化时还会触发一次 HMR** |

第二行那句 HMR 是它唯一的独有能力，值得单独读：**条件是"如果 Nitro 已经初始化"**，动作是"发出一个 HMR 事件去重载 Nitro 的 runtime config"。

实际含义：**在 dev 里用它，改动能被服务端那一侧立刻看到**；而在 Nitro 还没初始化（或者生产构建）的时候，**这条通知根本不会发生**，改动只是进了配置。页面没有再细分这两种时机各自的返回值。

## 三个页面没写的点

按本系列惯例逐条标注，别当成有依据：

- **合并方向**：原话只有 "This will be merged with the existing runtime configuration"，**没有说用哪种合并语义、也没有说谁优先**。同族的其他函数各自写明了（`updateAppConfig` 明说用 defu、`setGlobalHead` 明说你的值优先、`moduleDependencies` 的 `defaults` 与 `overrides` 各写方向），而本页没写。
- **返回类型为什么是联合**：`void | Promise<void>` 的两个分支各自在什么条件下出现，**未说明**。所以 `await updateRuntimeConfig(...)` 是安全写法（await 一个 `undefined` 不报错），但**不能假定它一定等到配置生效**。
- **`Record<string, unknown>` 的形状**：返回值没有 `public`/私有两侧的区分，**页面没有说明这份快照包含哪一侧、是否包含未公开的键**。要在模块里读私有值时，这一点必须先实测。

## 与保留命名空间的关系

runtime config 有一个不能随意占用的位置：**`runtimeConfig.app` 是保留命名空间**（示例里的键是 `baseURL`、`cdnURL` 这类），构建时报出来的那条要求自定义键放到 `public` 或顶层（服务端专用）。

这条与本页直接相关：如果你用 `updateRuntimeConfig({ app: { ... } })` 去改 `app` 下的自定义键，**它不是本页能担保的行为**，而是那条报错针对的形状。另外**会话相关的助手读的是 `runtimeConfig.appSecret`，用 `NUXT_APP_SECRET` 设置**，这一项还有"纯数字的密钥会被环境覆盖解析成 number"的坑（在运行时配置那一章有专段）。

## 什么时候用哪个

- 模块在 `setup` 里要读现值（比如按已有配置决定行为）：**本页的 `useRuntimeConfig()`**，注意它没有参数、返回的是未分侧的记录。
- 模块要给别人留可覆盖的默认值：**`nuxt.options.runtimeConfig.public.你的模块名` ＋ defu**（配方页那条路径）。
- 需要在 dev 里让服务端立刻拿到新值：**`updateRuntimeConfig`**，因为只有它写了 HMR。
- 要在服务端处理器里读配置：**不是本页**，按运行时的两套写法（传 event 或 `nuxt/server` 无参）。

下一篇讲模板那四个函数——`dependsOn` 是这一页唯一会让你"每次改动都重算"的字段。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/runtime-config
