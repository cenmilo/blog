---
url: /posts/b21326ed9dfccb8c5231cf279d3c7bb7/
title: 355 - dev 与 devServer：本地开发服务器的配置
date: 2027-02-07T10:00:00+08:00
updated: 2027-02-07T10:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt.config 里两个开发相关配置节。dev 是个布尔，表示"是否跑在开发模式"，官方明说"Normally, you should not need to set this"——由 nuxt dev 命令决定、你一般别手设。devServer 才是你常动的：它管本地开发服务器监听的 host、port（默认 3000）、是否 HTTPS（false，或给 key/cert）、CORS origin、loadingTemplate，以及一个不该直接设的 url（总被 dev server 覆盖）。这一篇把 devServer 各子项与默认值列清，并复述端口/host 的那条环境变量优先级（NUXT_PORT/NITRO_PORT/PORT/devServer.port）。

categories:
  - 前端开发

tags:
  - nuxt4
  - devServer
  - 端口
  - HTTPS
  - 开发配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`nuxt dev` 起的那个本地服务器，其行为由两个 nuxt.config 节控制：`dev` 和 `devServer`。名字像，一个你基本不碰、一个常碰。

## dev：一个你通常不该设的布尔

官方：

> Whether Nuxt is running in development mode. Normally, you should not need to set this.

- 类型 boolean，默认 `false`。
- 它表示"是不是开发模式"，正常由 `nuxt dev` 命令决定。**你不该在 nuxt.config 手设它**——它是给内部/特殊场景留的开关。知道它存在、别乱动即可。

## devServer：本地开发服务器的配置

这才是你按需求调的地方。它下面的子项（nuxt-config 的 `## devServer`）：

| 子项 | 类型 / 默认 | 作用 |
| --- | --- | --- |
| `host` | —— | 开发服务器监听地址 |
| `port` | number，默认 `3000` | 监听端口 |
| `https` | boolean，默认 `false` | 是否启用 HTTPS，可给 `{ key, cert }` |
| `cors.origin` | array | 开发服务器 CORS origin |
| `loadingTemplate` | function | 加载屏模板 |
| `url` | string，默认 `http://localhost:3000` | **别直接设**，总被 dev server 覆盖 |

两个要单独说：

- **https**：默认关。要给本地加 HTTPS，`devServer: { https: { key: './server.key', cert: './server.crt' } }`（官方示例）。本地证书信任那套（mkcert / NODE_EXTRA_CA_CERTS）是命令行 `--https` 的事，见 034/085。
- **url**：官方原话 "This should not be set directly as it will always be overridden by the dev server with the full URL (for module and internal use)."——它是给模块和内部用的，你设了也会被覆盖，别指望用它改访问地址。改端口用 `port` 或命令行。

## 端口和 host 到底谁说了算

真正起 `nuxt dev` 时，端口不是只看 `devServer.port`。完整优先级是一条环境变量链：

```
NUXT_PORT  ||  NITRO_PORT  ||  PORT  ||  devServer.port(默认 3000)
```

host 同理（`NUXT_HOST`/`NITRO_HOST`/`HOST` → devServer.host）。也就是说**环境变量压过 nuxt.config 的 devServer**——CI 或平台注入了 `PORT` 时，你 devServer.port 写多少都不算。这是"我配了端口却没生效"的常见原因。

## 顺带：devtools 与 devServerHandlers

同页紧挨着的两节，一句话坐标（细节各归各篇）：

- `## devtools`：开关 Nuxt DevTools（官方 note "Breaking changes for devtools might not reflect on the version of Nuxt."），命令行 `nuxt devtools enable/disable`（338）。
- `## devServerHandlers`：仅开发期的 Nitro 服务端 handler（329）。

## 自查

- dev 别手设，它表示是否开发模式、由 `nuxt dev` 决定。
- 调本地服务器：端口 `devServer.port`、host `devServer.host`、HTTPS `devServer.https:{key,cert}`；url 那项别碰（总被覆盖）。
- 端口/主机环境变量（NUXT_PORT/NITRO_PORT/PORT 等）会盖过 devServer——配了没生效先查是不是被注入了。
- devtools 与 devServerHandlers 是另两节，各看各篇。

## 下一篇

下一篇 356 串讲目录类配置：srcDir、serverDir、dir 与 buildDir——各自默认、基准目录，以及改名会牵动什么。

## 本文依据

- nuxt-config dev / devServer 节（`api/nuxt-config`）：dev（"Whether Nuxt is running in development mode. Normally, you should not need to set this."，boolean 默认 false）；devServer 子项 host（"Dev server listening host"）、port（默认 3000）、https（默认 false、`{key,cert}` 示例）、cors.origin（array）、loadingTemplate（function）、url（默认 `http://localhost:3000`、"should not be set directly as it will always be overridden"）；相邻 devtools 节与 devServerHandlers 节。
- devServer 环境变量优先级（NUXT_PORT/NITRO_PORT/PORT/devServer.port，见 085/034 记录）。
