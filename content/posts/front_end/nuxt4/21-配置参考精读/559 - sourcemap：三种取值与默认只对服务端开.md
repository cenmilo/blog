---
url: /posts/e020031e65dcdb4d0069a026d0a7cfb5/
title: 559 - sourcemap：三种取值与默认只对服务端开
date: 2027-04-16T10:00:00+08:00
updated: 2027-04-16T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`sourcemap` 一节把话说得很全。定性是**"Configures whether and how sourcemaps are generated for server and/or client bundles."**，并补一句**"If set to a single boolean, that value applies to both server and client."**——可以只写一个布尔同时管两端。三种取值各有官方定义：**`true` 生成并把源引用包含进最终产物、`false` 不生成、`'hidden'` 生成但产物里不含引用**，且 `'hidden'` 对服务端与客户端都可用。类型 `object`，**默认值是 `{"server": true, "client": false}`**。这也解释了线上最常见的现象：服务端堆栈能还原到源码，浏览器里的却是压缩位置。

categories:
  - 前端开发

tags:
  - nuxt4
  - sourcemap
  - 调试
  - 构建
  - 生产
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

同一个错误：服务端日志里的行号能点开看到源码，浏览器控制台里的却是一堆 `entry.BxY2.js:1:23456`。

这不是哪边坏了，是一个配置项的默认值长这样。

## 定义、两种写法、三个取值

`## sourcemap` 的三句：

> Configures whether and how sourcemaps are generated for server and/or client bundles.
>
> If set to a single boolean, that value applies to both server and client. Additionally, the `'hidden'` option is also available for both server and client.

然后是逐项定义：

| 取值 | 官方定义 |
| --- | --- |
| `true` | Generates sourcemaps and includes source references in the final bundle. |
| `false` | Does not generate any sourcemaps. |
| `'hidden'` | Generates sourcemaps but does not include references in the final bundle. |

- **类型**：`object`
- **默认**：

```json
{
  "server": true,
  "client": false
}
```

两点读法：**"whether and how"** 说明这个键同时管"要不要生成"和"生成的怎么用"；**`'hidden'` 两端都能用**（官方专门补了这句，容易被以为只属于某一边）。

`'hidden'` 这一档是配置项里少见的中间态：**产物旁边仍然有映射文件，但 bundle 里不留引用**。所以它适合"错误上报平台能把堆栈还原、但普通用户打开控制台看不到源码"这类用法——**不过这层用途是本系列对定义的读法，官方在这一节没有给使用建议或示例**。

## 两种写法的形状差异

由于"单个布尔会同时作用于两端"，两种写法都合法：

```ts
export default defineNuxtConfig({
  // or sourcemap: true
  sourcemap: {
    server: true,
    client: true,
  },
})
```

这正是调试页给的那份示例，注释里连单布尔写法都标了。**这份示例同时确证了三件事**：类型可以是对象、可以写 `true`、以及"想让客户端也有映射就显式把 `client: true` 写出来"。

要留意的是**默认值是对象形态**，`server`/`client` 两个键就是它的全部可配范围（本节没有第三个键）。

## 与相邻几处的关系

这一节在文档里不是孤岛，四处互相牵连：

**一、dev 的差异在别处说。** 调试页那句是 "Sourcemaps are enabled for your server build by default, **and for the client build in dev mode**"。而本节的默认写的是 `client: false`。**两句合起来才完整：列表默认是 `false`，开发模式另有安排**（这与 `webpack.cssSourceMap` 的"列表 `false`、说明句写 dev 为 true"是同一种写法风格）。

**二、CSS 的映射是另一个键。** `webpack.cssSourceMap`（默认 `false`，说明句 "defaults to `true` in development"）管的是 CSS，不在这个 `sourcemap` 键的控制范围内——**本节的措辞只说 "server and/or client bundles"**。

**三、Vite 侧还有一个入口。** `vite` 那节的官方示例在 `$server` 里写了 `build: { sourcemap: 'inline' }`。**同一件事在 Nuxt 层（`sourcemap`）与 Vite 层（`vite.$server.build.sourcemap`）各有一个写法，官方没有说明两者同时设时谁生效。**

**四、错误载荷是另一回事。** 服务端堆栈能还原，靠的是这节的 `server: true`；而客户端错误里能不能拿到原因，是 `error.cause` 那条规则（**生产环境 `cause` 不进错误响应也不进错误页载荷**）。别把"打开客户端 sourcemap"当成能拿到错误原因的办法。

## 一处文档没答的问题

本节只写"生成/包含引用"，没有写**映射文件落在哪、要不要上传到错误平台**。也没有一句关于**在生产环境暴露源码的风险提示**——`client: true` 会把源引用包含进最终 bundle，这一点从 `true` 的定义句能读出来（"includes source references in the final bundle"），**但会不会公开你的源码、要不要用 `'hidden'` + 平台上传，官方没给判断标准**。

所以能确证的边界是：**开客户端映射等于产物里带源码引用**；剩下的取舍按你自己的安全要求定，不是文档给的答案。

## 自查

- `sourcemap` 类型是 `object`、默认 `{"server": true, "client": false}`——**服务端默认有、浏览器默认没有**，这就是线上两种堆栈可读性不同的原因。
- 三个取值：`true` 生成并把源引用包含进产物、`false` 不生成、`'hidden'` 生成但不含引用；`'hidden'` 对两端都可用。
- 可以只写一个布尔同时管两端（官方明写），但默认值是对象形态，可配键只有 `server` 与 `client`。
- "客户端仅 dev 有映射"这件事写在调试页，本节列表默认是 `false`；CSS 的映射另有 `webpack.cssSourceMap`。
- `vite.$server.build.sourcemap` 与本键并存时谁生效，官方未说明；打开客户端映射意味着产物带源码引用，风险权衡文档没给标准。

## 下一篇

下一篇 560 读 `optimization` 那三个键：`asyncTransforms`（默认把 `defineNuxtPlugin` 与 `defineNuxtRouteMiddleware` 当 async 函数、三组 objectDefinitions）、`keyedComposables`（七项，各带 `argumentLength` 与 `source`）、`treeShake.composables`（client 与 server 两份默认清单，服务端那份摇掉十个 Vue 生命周期钩子）。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## sourcemap` 整节：定性句、单布尔作用于两端那句、`'hidden'` 两端可用那句、三种取值的逐条定义、Type `object` 与默认 JSON）
- https://nuxt.com/docs/4.x/guide/going-further/debugging（"服务端构建默认开启、客户端仅 dev"那句与 `sourcemap: { server: true, client: true }` 示例及其 `// or sourcemap: true` 注释）
- https://nuxt.com/docs/4.x/api/nuxt-config（`#### webpack.cssSourceMap` 的默认与 dev 说明句、`### vite.$server` 示例里的 `build.sourcemap: 'inline'`）
- https://nuxt.com/docs/4.x/getting-started/error-handling（生产环境错误载荷与响应里没有 `cause`，用于区分"映射可读"与"原因可得"）
