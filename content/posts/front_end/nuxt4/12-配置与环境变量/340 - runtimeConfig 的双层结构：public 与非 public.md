---
url: /posts/ca06f64e9be4bc89a7c5821870777413/
title: 340 - runtimeConfig 的双层结构：public 与非 public
date: 2027-02-02T10:00:00+08:00
updated: 2027-02-02T10:00:00+08:00
author: cmdragon

summary:
  摘要：P12 配置与环境变量从这里开始。runtimeConfig 的核心是一个两层结构：直接写在 runtimeConfig 下的键是私有的、只有服务端能读；写在 runtimeConfig.public 下的键会随页面 payload 下发、前后端都能读。为什么这么分——同一份配置既要能给后端用密钥、又要给前端用基址，用一层做不到。这一篇讲两层的语义、官方那个 apiSecret 与 public.apiBase 的对比例子、以及第三个被 Nuxt 保留的 app 命名空间（别往里加自己的键）。下一篇讲读写规则。

categories:
  - 前端开发

tags:
  - nuxt4
  - runtimeConfig
  - public
  - 私有配置
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（339）用一句话带过"配置放 runtimeConfig"。340-364 把这整块展开。第一篇先立结构：**runtimeConfig 不是一层，是"私有 + 公有"两层，外加一个 Nuxt 保留的 `app` 空间。**

## 官方那个对比例子

runtime-config 页给的第一个例子就把两层摆在一起：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    // The private keys which are only available within server-side
    apiSecret: '123',
    // Keys within public, will be also exposed to the client-side
    public: {
      apiBase: '/api',
    },
  },
})
```

读这张例子：

- `apiSecret` 直接写在 `runtimeConfig` 下 → **私有**，只有服务端能读。
- `apiBase` 写在 `runtimeConfig.public` 下 → **公有**，前后端都能读，官方说 "When adding `apiBase` to the `runtimeConfig.public`, Nuxt adds it to each page payload."（会进每一页的 payload）。

## 两层的分界线

nuxt-config 对 runtimeConfig 的定义句把分界说死（265、327 也引过）：

> It mainly should hold *private* configuration which is not exposed on the frontend. This could include a reference to your API secret tokens.
> Anything under `public` and `app` will be exposed to the frontend as well.

归纳成三条规则：

1. 顶层（`runtimeConfig` 直属的键）= 私有、只服务端。
2. `public` 下的键 = 会下发前端（进 payload）。
3. `app` 下的键 = 也会下发前端（但那是 Nuxt 自己用的，见下）。

"为什么非要两层"：一份配置常常既有"后端调外部服务要用的密钥"（绝不能到前端），又有"前端发请求要的接口基址"（必须到前端）。一层装不下这两种安全级别相反的东西，所以按"要不要给前端看"切开。

## 决定放哪一层的判据

问一句：**这个值让浏览器/用户看到，有没有坏处？**

- 有坏处（token、数据库连接串、任何 secret）→ 放顶层私有。前端 `useRuntimeConfig()` 里根本读不到它。
- 无坏处、且前端要用（API 基址、公开的功能开关）→ 放 `public`。

327 那个密钥代理接口的完整例子，正是"`apiSecret` 放顶层、handler 里读它拼进请求头、前端只看到 `/api` 的返回"。334 安全基线也建立在这条分界上。

## 第三个空间：app 是 Nuxt 保留的

除了私有和 `public`，还有 `app` 命名空间。use-runtime-config 页：

> Nuxt uses `app` namespace in runtime-config with keys including `baseURL` and `cdnURL`. ... This is a reserved namespace. You should not introduce additional keys inside `app`.

里面是 `app.baseURL`（默认 `/`）、`app.cdnURL` 这些 Nuxt 自己的挂载/资源路径（342 讲 `NUXT_APP_BASE_URL` 怎么覆盖）。它是"会下发前端"的那一类，但**是保留的**——你别往 `app` 里加自己的键，那会撞 Nuxt 的用途。

## 一图收束

| 放哪 | 谁能读 | 典型内容 |
| --- | --- | --- |
| `runtimeConfig.<key>`（顶层） | 只服务端 | secret、连接串、第三方 token |
| `runtimeConfig.public.<key>` | 服务端 + 前端（进 payload） | 接口基址、公开开关 |
| `runtimeConfig.app.*` | 服务端 + 前端，但 Nuxt 保留 | baseURL、cdnURL（别自加键） |

下一篇（341）讲读写规则的不对称：客户端 public 可写可响应式、服务端整个可读但只读。

## 自查

- 有 secret 性质的值放 runtimeConfig 顶层（只服务端读），前端读不到；前端要用的放 `public`。
- 判断标准就一句：这值给用户看到有没有坏处。
- `app` 是 Nuxt 保留命名空间，别往里加自己的键。
- `public` 的值会进每一页 payload（318/264），所以也别往 public 塞敏感东西。

## 下一篇

下一篇 341：useRuntimeConfig 读写规则：客户端改不了私有键

## 本文依据

- Runtime Config（`guide/going-further/runtime-config`）：定义例子（apiSecret 私有、public.apiBase 公有、"When adding apiBase ... Nuxt adds it to each page payload"）。
- nuxt-config runtimeConfig 节（`api/nuxt-config`）："It mainly should hold *private* configuration which is not exposed on the frontend ... Anything under `public` and `app` will be exposed to the frontend as well."
- useRuntimeConfig（`api/composables/use-runtime-config`）：note（server 需要的键直接放 runtimeConfig、两端都要的放 public）；`app` 保留命名空间（"You should not introduce additional keys inside `app`"、baseURL/cdnURL）。
