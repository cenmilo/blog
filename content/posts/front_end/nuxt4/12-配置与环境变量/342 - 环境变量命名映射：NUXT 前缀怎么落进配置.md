---
url: /posts/950ba17df9c4c501a5a029a640aaae45/
title: 342 - 环境变量命名映射：NUXT 前缀怎么落进配置
date: 2027-02-03T09:00:00+08:00
updated: 2027-02-03T09:00:00+08:00
author: cmdragon

summary:
  摘要：runtimeConfig 的键会被同名的 NUXT_ 环境变量在运行时覆盖（335），但"同名"有一套固定映射规则，写错一个字母就悄悄失效。这一篇把映射讲全：前缀必须是 NUXT_；下划线同时表示层级和大小写变化；public 层的键对应环境变量要插一段 PUBLIC（apiBase → NUXT_PUBLIC_API_BASE）；app.baseURL 对应 NUXT_APP_BASE_URL。再讲一个隐蔽坑：环境变量值会经 destr 自动转成 JS 类型，NUXT_MY_VAR=4848e0 会变成数字 4848，要保住字符串得把值本身用双引号包起来。

categories:
  - 前端开发

tags:
  - nuxt4
  - 环境变量
  - NUXT前缀
  - destr
  - runtimeConfig
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


335 说了"生产只认 `NUXT_` 前缀、运行时覆盖 runtimeConfig"。这一篇讲"名字到底怎么对上"——这是最容易写错、错了还悄无声息的一段。

## 两条前提

runtime-config 页把覆盖成立的条件列了两条：

> 1. Custom runtime config variables must be defined in your `nuxt.config`. This ensures that arbitrary environment variables are not exposed to your application code.
> 2. Only a specially-named environment variable can override a runtime config property. That is, an uppercase environment variable starting with `NUXT_` which uses `_` to separate keys and case changes.

第一条：**键得先在 `nuxt.config` 声明过**——没声明的 `NUXT_*` 不会自动进 runtimeConfig（这是防止任意环境变量泄进应用的护栏）。第二条：能覆盖的变量必须是**全大写、`NUXT_` 开头，用下划线同时分隔"层级"和"大小写变化"**。

## 映射规则与例子

把 runtimeConfig 的嵌套结构映射到环境变量名，规则是"每一层、每次驼峰转大写，中间用 `_`"：

| runtimeConfig 键 | 环境变量 |
| --- | --- |
| `apiSecret`（顶层） | `NUXT_API_SECRET` |
| `public.apiBase` | `NUXT_PUBLIC_API_BASE` |
| `app.baseURL` | `NUXT_APP_BASE_URL` |
| `app.cdnURL` | `NUXT_APP_CDN_URL` |

注意 `public` 这一层在名字里要**显式出现**成 `PUBLIC`——`apiBase` 想被覆盖，环境变量是 `NUXT_PUBLIC_API_BASE` 不是 `NUXT_API_BASE`。use-runtime-config 的 .env 例子就是这个：

```ini [.env]
NUXT_PUBLIC_API_BASE = "https://api.localhost:5555"
NUXT_API_SECRET = "123"
```

`app` 命名空间也是同一规则（340 说它保留），`NUXT_APP_BASE_URL`、`NUXT_APP_CDN_URL` 分别改挂载路径和 CDN。

## 一条隐蔽坑：值会被自动转成 JS 类型

这是本篇最想提醒的。runtime-config 那条 warning：

> Environment variable values are automatically cast to their JavaScript type using destr. For example, `NUXT_MY_VAR=4848e0` becomes the number `4848`. To keep a value a string, the environment variable value itself must contain literal double quotes: in a `.env` file, write `NUXT_MY_VAR='"4848e0"'` ...

含义：环境变量值不是"照字符串收下"，而是走一遍 destr 做类型推断。`4848e0` 看着像字符串，被当成科学计数法解析成了数字 `4848`。这类"我明明设的是字符串、读出来变数字/布尔"的怪事就是它。

要保住字符串，得把**值本身用双引号包起来**（`.env` 里 `NUXT_MY_VAR='"4848e0"'`），且引号要是值的一部分、别被 shell 吞掉。265 讲 appSecret 时那句"Nitro parses environment overrides, so a secret made only of digits arrives as a number"是同一个坑：一个纯数字的密钥可能被解析成 number 而不是 string。

## 别把默认值指向异名变量

一条容易做错的搭配（warning）：

> Setting the default of `runtimeConfig` values to differently named environment variables (for example setting `myVar` to `process.env.OTHER_VARIABLE`) will only work during build-time and will break on runtime.

别写 `apiBase: process.env.SOME_OTHER_NAME`——那只在构建期有效、运行时会断。正确做法是让环境变量名和 runtimeConfig 结构对得上（走上面那套 `NUXT_..._` 映射），交给覆盖机制。

## 自查

- 要覆盖某个 runtimeConfig 键：键先在 nuxt.config 声明；变量名走"全大写 + `NUXT_` 前缀 + 层级用 `_`"，`public` 层要带 `PUBLIC`（`NUXT_PUBLIC_API_BASE`）。
- 值会被 destr 转类型：数字串/`4848e0` 会变成 number；要当字符串就在 .env 里把值本身包双引号。
- 别把某个键的默认值指向"另一个名字"的 process.env——运行时会失效。
- app.baseURL/cdnURL 用 `NUXT_APP_BASE_URL`/`NUXT_APP_CDN_URL` 改，别往 app 里加自己的键。

## 下一篇
下一篇 343：构建期能过运行期会坏：默认值来源写错的坑

## 本文依据

- Runtime Config（`guide/going-further/runtime-config`）：两条前提（"must be defined in your nuxt.config"、"uppercase environment variable starting with NUXT_ which uses _ to separate keys and case changes"）；Example 的 `.env`（NUXT_API_SECRET / NUXT_PUBLIC_API_BASE）；destr 类型转换 warning（NUXT_MY_VAR=4848e0→4848、保住字符串要值里带双引号）；指向异名环境变量只构建期有效 warning；appSecret 那条纯数字被解析成 number（见 265）。
- useRuntimeConfig（`api/composables/use-runtime-config`）：.env 例子里的 `NUXT_PUBLIC_API_BASE`/`NUXT_API_SECRET`；`app.baseURL` 用 `NUXT_APP_BASE_URL`、`app.cdnURL` 用 `NUXT_APP_CDN_URL` 覆盖。
- .env（`directory-structure/env`）：生产只认 NUXT_ 前缀（见 335）。
