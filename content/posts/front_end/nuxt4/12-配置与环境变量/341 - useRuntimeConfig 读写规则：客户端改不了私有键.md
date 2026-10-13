---
url: /posts/e59a9a8628a85ebd8c94e12673890a74/
title: 341 - useRuntimeConfig 读写规则：客户端改不了私有键
date: 2027-02-02T11:00:00+08:00
updated: 2027-02-02T11:00:00+08:00
author: cmdragon

summary:
  摘要：useRuntimeConfig() 在服务端和客户端能读到的东西、能不能改，是不对称的。客户端只有 runtimeConfig.public（和 Nuxt 内部的 app）可读，而且那份是可写、可响应式；服务端整个 runtimeConfig 可读，但只读——官方理由是避免上下文共享。这一篇把这个不对称讲清，落到用法：想改配置只能在客户端改 public；服务端读密钥要用 useRuntimeConfig(event) 拿到被环境变量覆盖过的那份；模板里读公有配置可用 $config.public。顺带纠正一个常见误解——"客户端能写"不等于"改了会影响服务端"，两端各是各的实例。

categories:
  - 前端开发

tags:
  - nuxt4
  - useRuntimeConfig
  - 读写
  - public
  - 响应式
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（340）讲了 runtimeConfig 的分层。这一篇讲读它的 `useRuntimeConfig()` 本身有个不对称：服务端和客户端能读的范围、能不能改，都不一样。

## 核心：两端的读写不对称

runtime-config 页把这条 important 写得很清楚：

> The behavior is different between the client-side and server-side:
> - On client-side, only keys in `runtimeConfig.public` and `runtimeConfig.app` (which is used by Nuxt internally) are available, and the object is both writable and reactive.
> - On server-side, the entire runtime config is available, but it is read-only to avoid context sharing.

拆成两栏：

| | 客户端 | 服务端 |
| --- | --- | --- |
| 能读什么 | 只有 `public` + `app` | 整个 runtimeConfig（含私有） |
| 能不能写 | 可写 | **只读**，理由是 "to avoid context sharing" |
| 是不是响应式 | 是（writable and reactive） | —— |

两条要点值得单拎：

- **客户端读不到私有键**：`config.apiSecret` 在浏览器是拿不到的（340 的分层在读取端兑现）。
- **服务端为什么只读**：服务端是多请求共享同一进程跑的（250 那条串数据同源），如果允许改这份全局 config，就会在不同请求间串起来。所以服务端刻意只读。反过来客户端那份可写可响应式，因为它是这个用户自己的实例。

## 基本读法

官方两端的读法：

```vue [app/app.vue]
<script setup lang="ts">
const config = useRuntimeConfig()
</script>
```

```ts [server/api/foo.ts]
export default defineEventHandler((event) => {
  const config = useRuntimeConfig(event)
})
```

服务端那条**传了 `event`**。这不是可选项的差别——runtime-config 那页 note：

> Giving the `event` as argument to `useRuntimeConfig` is optional, but it is recommended to pass it to get the runtime config overwritten by environment variables at runtime for server routes.

传 `event` 才能拿到"运行时被 `NUXT_` 环境变量覆盖过"的那份 config（335 的覆盖链路）；不传可能读到构建期的默认值。327 那个密钥例子用的正是 `useRuntimeConfig(event)`。

## 模板里读公有配置

不必每个组件都调 `useRuntimeConfig()`——公有配置在模板里有个现成入口。tip：

> Public runtime config is accessible in Vue templates with `$config.public`.

模板里直接 `{{ $config.public.apiBase }}` 就行。注意是 `$config.public`——`$config` 上挂的也是"只有 public 可读"的那个客户端视图。

## 一个常见误解要纠正

"客户端可写"容易被读成"我改了前端 config，服务端/别人也跟着变"——不会。两端是各自的实例（服务端每请求一份、客户端每浏览器一份），客户端改 `config.public.xxx` 只影响当前这个浏览器里的响应式值，不回传服务端、也不影响别的用户。真要"运行时可改、又给前端"的配置，那是 `app.config` + `updateAppConfig` 的活（266），不是 runtimeConfig。

## 自查

- 客户端只读得到 `public`/`app`、且那份可写可响应式；服务端整个可读但只读（防上下文共享）。
- 服务端路由里读 config 用 `useRuntimeConfig(event)`，传 event 才拿到被环境变量覆盖过的值。
- 模板里读公有配置用 `$config.public`，不用每个组件都调组合式。
- 别把"客户端可写"理解成会影响服务端——两端各自实例；要前端可改的配置走 app.config。

## 下一篇

下一篇 342 讲环境变量到 runtimeConfig 键的命名映射：NUXT_ 前缀、大小写转下划线、public 层要加 PUBLIC，以及 destr 把值当类型转换带来的"数字串变数字"坑。

## 本文依据

- useRuntimeConfig（`api/composables/use-runtime-config`）：读法两端示例（app.vue `useRuntimeConfig()`、server/api `useRuntimeConfig(event)`）、Define Runtime Config 例子、Access 例子里 config.public.apiBase 与 config.apiSecret（后者 only server-side）；tip "Public runtime config is accessible in Vue templates with $config.public."
- Runtime Config（`guide/going-further/runtime-config`）：两端读写不对称 important（client 只 public/app 可写可响应式、server 全部可读但只读"to avoid context sharing"）；服务端路由传 event 的 note（见 335/327）。
- app.config / updateAppConfig（`guide/concepts`/`api`）：前端可改配置走 app.config（见 265/266）。
