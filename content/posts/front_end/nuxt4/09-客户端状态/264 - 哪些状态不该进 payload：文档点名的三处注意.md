---
url: /posts/13f9db90de28a5cd9ee479ee2d42b746/
title: 264 - 哪些状态不该进 payload：文档点名的三处注意
date: 2027-01-08T09:00:00+08:00
updated: 2027-01-08T09:00:00+08:00
author: cmdragon

summary:
  摘要：useState 能过水合，靠的是把值序列化、随 HTML 一起发给浏览器（这段内联数据叫 payload）。便利的另一面是：payload 和客户端 JS 包都是用户能直接看到的明文。文档在三个地方各自点了一句"这里有东西不该放"——app.config 别放密钥（它进客户端包）、runtimeConfig 私有键别渲染出来或传给 useState、useState 本身只能放可 JSON 序列化的东西。这一篇把这三句原话摆到一起，讲清"进 payload / 进客户端包"到底意味着什么，以及密钥该待在哪一侧。

categories:
  - 前端开发

tags:
  - nuxt4
  - payload
  - runtimeConfig
  - 安全
  - useState
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


SSR 应用首屏 HTML 里，除了你看见的文字，还藏着一段给浏览器"接手用"的数据。它通常长这样：一个 `<script type="application/json">`，里面是本次渲染用到的状态和取数结果。这段东西，Nuxt 里叫 **payload**。251 篇说 `useState` "能过水合"，原理就是把值序列化塞进这段 payload，浏览器再读回来。

问题也在这儿：**payload 是发给浏览器的，用户在 DevTools 里能直接翻到明文。** 同样，客户端 JS 包（浏览器要下载执行的那堆代码）也是用户能看到的。文档在三个不相邻的地方，各自点了一句"这两样里别放敏感东西"。把它们凑到一起看，才知道哪些状态天生不能进 payload。

## 注意一：app.config 会进客户端包

`app.config.ts` 是用来放"响应式配置"的（265 篇展开）。但它有一句 caution，级别不低：

> Do not put any secret values inside `app.config` file. It is exposed to the user client bundle.

原因是 `useAppConfig()` 能在浏览器里读，所以这个文件的内容会被打进客户端包、随 JS 发到用户手上。它是配置容器，不是密钥保险箱。想要"只在服务端可见"的密钥，不是放这儿。

## 注意二：runtimeConfig 的私有键，别渲染也别塞进 useState

真正的密钥待处是 `runtimeConfig`。`nuxt.config` 那一页对它的定位是：

> It mainly should hold *private* configuration which is not exposed on the frontend. This could include a reference to your API secret tokens.
> Anything under `public` and `app` will be exposed to the frontend as well.

也就是说 `runtimeConfig` 分两半：裸写的键（如 `apiSecret`）只在服务端可读；放在 `public`、`app` 下面的键会下发给前端。这个"服务端有全部、客户端只有 public"的分工，runtime-config 指南里写得更硬：

> - On client-side, only keys in `runtimeConfig.public` and `runtimeConfig.app` ... are available, and the object is both writable and reactive.
> - On server-side, the entire runtime config is available, but it is read-only to avoid context sharing.

那把私钥怎么就会泄出去？靠"手滑把它带到前端"。指南给了一句 caution，正好点出两条泄漏路径：

> **Security note:** Be careful not to expose runtime config keys to the client-side by either rendering them or passing them to `useState`.

两条路都通到"用户看得见"：一条是直接把值渲染进模板（出现在 HTML 里），另一条是**把私有键的值赋给 `useState`**——而 `useState` 会进 payload。这就是本篇标题的落点：`useState` 是个"会广播到浏览器"的容器，所以从 `runtimeConfig` 往 `useState` 里搬数据之前，先确认搬的是 `public` 的东西，不是私钥。

服务端 handler 里读密钥、只把结果返回给前端，是安全写法；把 `apiSecret` 本身赋进一个 `useState`，等于当着用户面把密钥打印在 HTML 里。

## 注意三：useState 只能放能序列化成 JSON 的东西

这一条不是"别放密钥"，而是"放错了会直接报错/变形"。useState 页的 important：

> Because the data inside `useState` will be serialized to JSON, it is important that it does not contain anything that cannot be serialized, such as classes, functions or symbols.

state-management 那页对同一句又强调了一遍。含义是：既然要进 payload，就得先 `JSON.stringify` 得动。类实例、函数、Symbol 这些序列化不了，硬塞进去要么在 SSR 阶段炸（useState 排错那节提到 `Cannot stringify arbitrary non-POJOs`，正解是用 `definePayloadPlugin` 加自定义序列化器），要么过去之后变成残缺的空壳。

这一条和"性能/体积"也连得上。payload 是随 HTML 首屏发出去的，塞一大坨不必要的数据，等于给每个首屏请求都加重量。所以能不进 payload 的数据就别进：`useAsyncData`/`useFetch` 有个 `serialize: false`（服务端取到但不写进 payload），实验开关 `stripNeverHydratedData` 则专门把"永不水合组件"的数据从 payload 里剔掉——这些手段的动机和"少放敏感/大块数据"是同一件事（267 篇细讲）。

## 谁放哪儿：一张对照

| 数据 | 该放哪 | 会不会被用户看到 |
| --- | --- | --- |
| API 私钥、数据库口令 | `runtimeConfig` 顶层私有键（只在服务端读） | 不会，前提是你不渲染它、不塞进 useState |
| 前端要用的配置（接口基址等） | `runtimeConfig.public` / `app.config` | 会，这是设计如此 |
| 一次加载内共享的 UI 状态 | `useState` | 会进 payload，别放敏感值，且必须能 JSON 序列化 |
| 会话 / 登录态 | 加密封装的 cookie（见 259） | cookie 本身可见，但会话由模块加密 |

## 自查

- 有没有把 `runtimeConfig` 的私有键渲染进模板，或者赋给某个 `useState`？这是最典型的两种泄漏。
- `app.config` 里是不是混进了密钥？它进客户端包，不放敏感值。
- `useState` 里塞的东西能不能被 `JSON.stringify`？有类实例/函数就换成纯数据，或用 `definePayloadPlugin`。
- 首屏 payload 是不是带了大块不必要数据？考虑 `serialize: false` 或 `stripNeverHydratedData`。

## 下一篇

下一篇 265 专门讲两个配置容器——`app.config` 与 `runtimeConfig`——到底怎么分工：一个响应式、进客户端包；一个分公私两半、私有只在服务端。
