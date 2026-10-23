---
url: /posts/7172b7d09a7c9feaac08bff3805337d8/
title: 339 - 服务端小结：Nitro 能力地图
date: 2027-02-02T09:00:00+08:00
updated: 2027-02-02T09:00:00+08:00
author: cmdragon

summary:
  摘要：P11 到这里收尾，用一张地图把服务端这一层收纳干净。地基是 Nitro 引擎（h3 handler 模型、独立于 node_modules 的 .output、15+ preset）。往下分五块：写接口的 server 目录（api/routes/middleware/plugins/utils/types 各管什么）、按路径定行为与缓存的 routeRules（prerender/swr/isr/ssr/redirect/cors/headers/noScripts/appMiddleware）、配置与密钥的 runtimeConfig + .env 链路、取数一致性（$fetch 直调、useFetch/useRequestFetch、baseURL）、以及排错。每块标回它讲在哪一篇，方便你日后定位。

categories:
  - 前端开发

tags:
  - nuxt4
  - Nitro
  - 服务端小结
  - routeRules
  - 能力地图
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


300-339 从 Nitro 讲到了服务端排错，零件够了。这一篇把它们收成一张地图，日后遇到服务端的问题，先在这张图上找到那块，再翻对应篇。

## 地基：一切跑在 Nitro 上

官方对 Nuxt 服务端的定调：

> Nuxt's server is Nitro. ... Nuxt's server is powered by a new server engine: Nitro.

它给 Nuxt 三样超能力（getting-started/server 原话）：

> - Full control of the server-side part of your app
> - Universal deployment on any provider (many zero-config)
> - Hybrid rendering

底下是 h3（"a minimal H(TTP) framework built for high performance and portability"）。产物是独立于 node_modules 的 `.output`、靠 preset 适配 15+ 平台（300、331）。**记住这一层，很多"为什么服务端能这样"的答案都是"Nitro 提供的"。**

## 五块能力，逐块标篇

### 一、写接口：server 目录

六个子目录各管一件事（301、302）：

| 目录 | 作用 | 关键 |
| --- | --- | --- |
| `server/api/` | 数据接口 | 自动带 `/api` 前缀 |
| `server/routes/` | 服务端路由 | 不加前缀 |
| `server/middleware/` | 服务端中间件 | 每请求跑、不该 return（326） |
| `server/plugins/` | Nitro 插件 | `defineNitroPlugin`、订服务端钩子（287） |
| `server/utils/` | 服务端 helper | 服务端上下文自动导入（311） |
| `server/types/` | 服务端类型 | 只扫顶层、仅服务端自动导入（301） |

handler 一律 `defineEventHandler`、`return` 值即 JSON（304），方法用后缀绑（303），动态段 `[id]`/`[...slug]`（306）。

### 二、按路径定行为：routeRules

声明式一张表，机制是"自动注册中间件 + 用 Nitro 缓存层包住路由"（325）。服务端侧选项：redirect、ssr、cors、headers、swr、isr、prerender、noScripts、appMiddleware（325），外加 cache（getting-started/server 示例）与 proxy（这一节没逐项列）。`nuxt generate` 下混合渲染不可用。

### 三、配置与密钥：runtimeConfig + .env

一条链路（335、327）：`.env` 只在 dev/build/generate 读 → 键在 nuxt.config 的 runtimeConfig 声明 → 应用用 `useRuntimeConfig()` 读。生产不读 .env、只认 `NUXT_` 前缀变量在运行时覆盖。私密密钥放顶层私有键、只服务端读、不进包不进 payload。

### 四、取数一致性：$fetch / useFetch

SSR 服务端 `$fetch` 直调 handler、不自动带浏览器头（300、318）；`useFetch` 底层用 `useRequestFetch` 自动转发上下文和头；`useRequestHeaders` 精挑转发。相对 `/api` 两端一致，baseURL/子路径要实测（332）。

### 五、跑起来之后：生命周期与排错

一次初始请求七步（319：Nitro 插件→服务端中间件→应用插件→validate→路由中间件→页面取数→HTML 输出）；排错靠 sourcemap/`--inspect`/`debug:true`/DevTools（338）。

## 一句总纲

把 300-339 压成一句：**应用代码在 `app/`、服务端代码在 `server/`、两者不许互 import（313/312）；服务端跑在 Nitro/h3 上、按 routeRules 定行为、按 runtimeConfig 拿配置、产物 `.output` 靠 preset 部署。** 遇到服务端的事，先定它落在上面五块里的哪一块。

## 自查（这层你该能答上的题）

- 一个接口要带 `/api` 前缀还是不带？放哪个目录？（302）
- 想给一段路径加缓存/做重定向/只客户端渲染，在哪配？（routeRules，325）
- 第三方密钥能不能出现在前端 bundle？正确放哪？（runtimeConfig 私有键，327）
- SSR 里取自家接口会不会重复请求、要不要手动带 cookie？（300/318）
- 服务端代码报错，第一反应调什么工具？（338）

能答上，服务端这层就通了。340-364 从这里接上，专讲配置与环境变量的完整体系。

## 下一篇

下一篇 340（340-364 配置与环境变量开局）讲 runtimeConfig 的双层结构：public 与非 public 各暴露给谁、为什么这么分。
