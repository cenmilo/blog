---
url: /posts/a02eb23905b93638b041a116c14c1f05/
title: 348 - runtimeConfig 与 app.config：官方那张对比表
date: 2027-02-05T09:00:00+08:00
updated: 2027-02-05T09:00:00+08:00
author: cmdragon

summary:
  摘要：runtimeConfig 和 app.config 都往应用暴露值，官方 configuration 页给了张七行对比表来区分。选型先记两句判据：runtimeConfig 放"要在构建后用环境变量指定的公私 token"，app.config 放"构建期就定的公开配置（主题、标题等不敏感的）"。表里七行分别是：客户端如何拿到（Hydrated 水合 vs Bundled 打包）、能否被环境变量覆盖、是否响应式、类型支持（runtimeConfig 部分、app.config 完整）、能否按请求配置、HMR、能否放非基本 JS 类型。这一篇逐行讲这些差别在实操里意味着什么。

categories:
  - 前端开发

tags:
  - nuxt4
  - runtimeConfig
  - app.config
  - 选型
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


340 讲了 runtimeConfig 的内部分层。这一篇讲**它和 app.config 之间怎么选**——官方 configuration 页有专门一节和一张表。

## 两句选型判据

官方先给了两条判断句：

- `runtimeConfig`：Private or public tokens that need to be specified after build using environment variables.（要在构建后用环境变量指定的公私 token）
- `app.config`：Public tokens that are determined at build time, website configuration such as theme variant, title and any project config that are not sensitive.（构建期就定、不敏感的公开配置，如主题、标题）

一句概括：**要构建后改、或含密钥 → runtimeConfig；构建期定死、纯前端用的非敏感配置 → app.config。**

## 官方对比表逐行

configuration 页的表原文：

| Feature | runtimeConfig | app.config |
| --- | --- | --- |
| Client-side | Hydrated | Bundled |
| Environment variables | ✅ Yes | ❌ No |
| Reactive | ✅ Yes | ✅ Yes |
| Types support | ✅ Partial | ✅ Yes |
| Configuration per request | ❌ No | ✅ Yes |
| Hot module replacement | ❌ No | ✅ Yes |
| Non-primitive JS types | ❌ No | ✅ Yes |

逐行翻译成人话：

1. **Client-side：Hydrated vs Bundled**。runtimeConfig 的 public 值会进页面 payload、水合时接过来（Hydrated，见 264 payload）；app.config 直接被打进客户端包（Bundled，所以 264 说它别放密钥——它在包里是明文）。
2. **Environment variables**：runtimeConfig 能被 `NUXT_` 变量覆盖（342）；app.config **不能**（构建期定死，345）。
3. **Reactive**：两者都响应式（app.config 用 `updateAppConfig` 改，266；runtimeConfig 客户端 public 那份可写且响应式，341）。
4. **Types support**：runtimeConfig 类型是**部分**（Partial），app.config 是**完整**。这是 runtimeConfig 的一个已知短板（344 的类型化就是在补它）。
5. **Configuration per request**：runtimeConfig 不行（它是进程/构建级 + 环境变量级），app.config 能**按请求**给不同配置——官方 app-config 那一节说 `useAppConfig()` 在服务端"as a copy per request when passed the event"（265/313 那条，app.config 随请求可不同）。
6. **Hot module replacement**：runtimeConfig 改了要重启进程（335 dev 改 .env 触发 restart），app.config 支持 HMR 热改（266 那句 "editing it with HMR"）。
7. **Non-primitive JS types**：runtimeConfig 因为要序列化进 payload（264），只能放可序列化值、放不下函数/类实例等；app.config 能放非基本 JS 类型（它进客户端包、不走 payload 序列化那套约束）。

## 什么时候用哪个

| 需求 | 选 |
| --- | --- |
| API 密钥、只服务端读 | runtimeConfig 私有键 |
| 部署后才能定的接口基址（分环境） | runtimeConfig.public + NUXT_ 变量 |
| 主题、站点标题、不敏感的前端开关 | app.config |
| 要放函数/非序列化对象、要 HMR 热改 | app.config |
| 要按请求返回不同的配置 | app.config |

## 自查

- 先问两句：要构建后改或含密钥吗（→runtimeConfig）？纯前端、构建期定、不敏感吗（→app.config）？
- 记住 runtimeConfig 的短板那几行：类型只 Partial、不能按请求、无 HMR、放不下非基本类型；app.config 反过来但不能被环境变量覆盖、且值打进客户端包（别放密钥）。
- 两者都响应式、都能前端读，但读到的路径不同：runtimeConfig 靠水合进 payload、app.config 直接进包。

## 下一篇

下一篇 349 专讲 .env 与密钥：文档给的存法、生产不读 .env、以及哪些值绝不能进 .env 进而被 runtimeConfig 下发前端。
