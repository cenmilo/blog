---
url: /posts/7b1dfb2aa0c9f7c1a77caabf9d93a964/
title: 587 - devServer：六项里只有一项记载完整，其余各缺一块
date: 2027-04-25T11:00:00+08:00
updated: 2027-04-25T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## devServer` **自己连定性句都没有**，直接进六个子项。按"类型行／默认值行／示例／说明句"清点，**只有 `port` 一项四样齐全**（`number` ＋ `3000`）；`host` 只有一句 "Dev server listening host"，**没有类型也没有默认值**；`cors.origin` 的默认值是 `[{}]`（一个空对象组成的数组，含义未解释）；`loadingTemplate` 写着 `function` 却没有参数说明；`https` 的类型行是 `boolean`，**而官方示例给它的是一个对象 `{ key: './server.key', cert: './server.crt' }`**；`url` 有默认值 `"http://localhost:3000"`，同时被写明**"不应该直接设置"**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 开发服务器
  - HTTPS
  - 跨域
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

本地要换端口、要让手机访问同一台机器上的开发服务、要上 HTTPS——这些都在 `devServer` 下面。但这一节的六个子项，**记载完整度差异大到会让人误以为是不同的作者写的**。

## 六项清点

| 子项 | 说明句 | Type | Default | 示例 |
| --- | --- | --- | --- | --- |
| `cors` | "Set CORS options for the dev server" | 无 | 无 | 无 |
| `cors.origin` | **无说明句** | `array` | `[{}]` | 无 |
| `host` | "Dev server listening host" | **无** | **无** | 无 |
| `https` | "Whether to enable HTTPS." | `boolean` | `false` | **有（但给的是对象）** |
| `loadingTemplate` | "Template to show a loading screen" | `function` | 无 | 无 |
| `port` | "Dev server listening port" | `number` | `3000` | 无 |
| `url` | "Listening dev server URL." ＋ 一句禁令 | `string` | `"http://localhost:3000"` | 无 |

（`## devServer` 本身**没有定性句**——标题下面直接是 `### cors`。）

四样齐全只有 `port` 一项。下面三处缺口最实际。

## https：类型行写布尔，示例给对象

说明句是布尔口径（"Whether to enable HTTPS."），类型行 `boolean`、默认 `false`。可官方示例是这样的：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  devServer: {
    https: {
      key: './server.key',
      cert: './server.crt',
    },
  },
})
```

**`key` 与 `cert` 两个字段只在这一处出现过**，配置参考没有为它们建子项，`## devServer` 下也没有 `### https.key` 之类。**这两个路径相对什么解析，本节没写**（示例字面上是相对项目根目录的 `./server.key`）。

这与上一篇 `typescript.typeCheck` 是同一种缺口：类型行只写 `boolean`，正文与示例却给了第三种形状。**读类型行会漏掉对象写法。**

## url：给了默认值，又写着不要设它

`url` 的默认值是 `"http://localhost:3000"`，紧跟一句限制：

> This should not be set directly as it will always be overridden by the dev server with the full URL (for module and internal use).

这句要按字面执行：**你设的值会被开发服务器用完整 URL 覆盖掉**，这一项是给模块与内部用的。所以"改本地访问地址"这件事**不能靠写 `devServer.url` 实现**——能设的只有 `host` 与 `port`。

而 `host` 这一项恰好是全节记载最薄的：**只有一句 "Dev server listening host"，没有类型行、没有默认值、没有示例**。它接受什么形状（`'0.0.0.0'`？`true`？）、不写时监听哪里，**本节都没写**。

## cors.origin：默认值是一个空对象组成的数组

`cors` 那句是 "Set CORS options for the dev server"，它下面只有一个子项 `origin`，类型 `array`，默认值原文：

```json
[
  {}
]
```

**这是全节最难读的一个默认值**：数组里装着**一个空对象**，而这个空对象有哪些字段、能填什么，**文档一个字都没有**。官方也没有给 `cors` 的任何示例，所以"要放行某个来源该怎么写"在本节没有依据——**只能实测**。

`loadingTemplate` 同样是形状未定义的那一类：类型 `function`、说明只有一句 "Template to show a loading screen"，**参数列表、返回类型、以及它与 `spaLoadingTemplate`（生产/SPA 加载屏）的分工，本节都没写**。

## 自查

- `## devServer` 无定性句；六项里**只有 `port` 记载完整**（`number` ＋ `3000`）。
- `host`：一句话，**无类型无默认无示例**。
- `https`：说明句与类型行是布尔口径（默认 `false`），**官方示例却设对象 `{ key, cert }`**；两字段无子项、路径基准未写。
- `url`：默认 `"http://localhost:3000"`，**文档明写不该直接设它**，会被开发服务器覆盖；改访问地址靠 `host` 与 `port`。
- `cors.origin`：`array`，默认 `[{}]`，**空对象里能填什么无记载、无示例**。
- `loadingTemplate`：`function`，**参数与返回未写**，与 `spaLoadingTemplate` 的分界不在本节。

## 下一篇

下一篇 588 看隔壁的 `## dev`：它只有三行（"Whether Nuxt is running in development mode."、"Normally, you should not need to set this."、`boolean` 默认 `false`），**而且整套 4.x 里没有任何一处示范手动设它**——这一键在模块与构建器相关页里是被**读**的那个（`nuxt.options.dev`），不是被写的。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## devServer` 无定性句的事实；`### cors`、`#### cors.origin` 的类型与默认值 `[{}]`、`### host` 一句、`### https` 说明句＋`boolean`＋`false`＋对象示例、`### loadingTemplate` 一句＋`function`、`### port` 两句、`### url` 默认值与"should not be set directly"整句）
