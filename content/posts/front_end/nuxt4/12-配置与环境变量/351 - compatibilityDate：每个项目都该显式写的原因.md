---
url: /posts/4e4de8a0e4d40dce7467d43589d2c5ec/
title: 351 - compatibilityDate：每个项目都该显式写的原因
date: 2027-02-06T09:00:00+08:00
updated: 2027-02-06T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：compatibilityDate 是 nuxt.config 里一个日期字符串，决定你的项目用哪一批"行为默认值"。官方给的理由很实在：一些 preset（Nitro、Nuxt Image 等）会在不打大版本号的情况下改变行为，用这个日期把项目钉在"你上次确认过的那套默认"上，升级 Nuxt/Nitro 时就不会被悄悄换掉默认。没设它对应错误码 B5001，解法是填当天日期（比如 compatibilityDate: 'YYYY-MM-DD' 换成真实今天）来 opt 进当前默认。这一篇讲它的机制、为什么别省、以及它和 future/compatibilityVersion 的区别。

categories:
  - 前端开发

tags:
  - nuxt4
  - compatibilityDate
  - B5001
  - 默认值
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

nuxt.config 里有一个 `compatibilityDate` 字段，很多 starter 项目会带上、也有人不写。这一篇说清它是干什么的、为什么官方希望你写。

## 它是什么

nuxt-config 的定义：

> Specify a compatibility date for your app. This is used to control the behavior of presets in Nitro, Nuxt Image and other modules that may change behavior without a major version bump.

B5001 那条错误码把"为什么需要它"讲得更透：

> No `compatibilityDate` is set in your Nuxt configuration. Nuxt uses this date to decide which behaviour defaults to apply, so your project stays stable across Nuxt and Nitro updates instead of silently picking up new defaults.

合起来：它是**一个日期，用来决定"我该套哪批行为默认"**。关键在于——有些默认行为（Nitro 的 preset、Nuxt Image，以及其它模块）**不会等到大版本更新就改变**。你设了 compatibilityDate，Nuxt 就按"截至那天"的默认来，而不是每次升级都悄悄给你换新的。官方也说还在改进这块的工具（"We plan to improve the tooling around this feature in the future."）。

## 为什么别省

不写它不是"没影响"，而是"默认值会随依赖升级漂移"。你线上跑得好好的，某天升级 Nitro 的一个小版本，某个 preset 的默认行为变了，你没改一行代码，行为却变了——compatibilityDate 就是防这个的：把项目钉在一个确定的默认集上。

所以每条 starter、每个新项目都该显式写一句（016 的 starter 骨架里就有它）。

## 怎么写

B5001 的 Resolution 直接给了：

> Add `compatibilityDate` to your `nuxt.config`, using today's date so you opt in to the current defaults:

```ts
export default defineNuxtConfig({
  // use the current date, e.g. the day you add this line
  compatibilityDate: 'YYYY-MM-DD',
})
```

`'YYYY-MM-DD'` 是占位符，换成你添加这行的**当天真实日期**（ISO 形式，如 `'2026-10-24'`）。官方口径是"用今天的日期"——意思是你今天确认过当前默认没问题，就钉在今天。将来主动升级、验证新默认后，再把这个日期往后推。

## 和 future / compatibilityVersion 的区别

三个都和"版本/默认"有关，别混（下一篇 352 讲 future）：

| 字段 | 管什么 |
| --- | --- |
| `compatibilityDate` | 一个**日期**，锁"截至那天的行为默认"，防小版本升级悄悄漂移 |
| `future.compatibilityVersion`（如 5） | 一个**主版本号**，整套切到 Nuxt 5 的默认（提前体验下一大版本） |
| `experimental.*` | 单个**实验特性**的开/关 |

compatibilityDate 是"稳定当下"，compatibilityVersion 是"提前跨大版本"，两者解决不同问题。

## 自查

- 你的 nuxt.config 里有没有 compatibilityDate？没有就先补一句当天日期，别让它靠默认漂移。
- 它是 ISO 日期字符串，值写"你确认过当前默认的哪天"。
- 别把它和 `future.compatibilityVersion` 搞混：前者锁默认防漂移、后者跨大版本。
- 主动升级 Nuxt/Nitro 并验证后，再考虑推进这个日期。

## 下一篇

下一篇 352 讲 future 节：compatibilityVersion、multiApp、typescriptBundlerResolution 这三个提前启用未来默认行为的开关。

## 本文依据

- nuxt-config compatibilityDate 节（`api/nuxt-config`）："Specify a compatibility date for your app."、"This is used to control the behavior of presets in Nitro, Nuxt Image and other modules that may change behavior without a major version bump."、"We plan to improve the tooling around this feature in the future."
- B5001（`errors/b5001`）："No compatibilityDate is set ... Nuxt uses this date to decide which behaviour defaults to apply, so your project stays stable across Nuxt and Nitro updates instead of silently picking up new defaults."；Resolution "using today's date so you opt in to the current defaults" + `compatibilityDate: 'YYYY-MM-DD'`。
- features / future（`guide/going-further/features`）：compatibilityVersion（见 352）。
