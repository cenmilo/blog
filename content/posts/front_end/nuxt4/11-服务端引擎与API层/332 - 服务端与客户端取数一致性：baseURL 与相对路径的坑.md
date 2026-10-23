---
url: /posts/647f81482c5c54462767d483f7bc688a/
title: 332 - 服务端与客户端取数一致性：baseURL 与相对路径的坑
date: 2027-01-30T11:00:00+08:00
updated: 2027-01-30T11:00:00+08:00
author: cmdragon

summary:
  摘要：同一个取数调用，SSR 阶段在服务端跑、水合后在浏览器跑，两端的"地址怎么解析"不一样，是 baseURL 类 bug 的根源。这一篇把一致性讲清：相对路径 `/api/...` 在两端都指向你自己的应用（SSR 阶段服务端 $fetch 直调那个 handler、还会经 useRequestFetch 把头带过去），而绝对 URL（`http://…`）在 SSR 就是真打外部服务。baseURL 有三层（请求选项、runtimeConfig.public、app 命名空间）；子路径部署时相对路径是否带 app.baseURL，文档未写、要实测。给一条排查判据：两端取到不一样 → 看是不是用了绝对 URL 或 baseURL 只在某一层设了。

categories:
  - 前端开发

tags:
  - nuxt4
  - baseURL
  - 取数一致性
  - SSR
  - useFetch
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


一个 `useFetch`/`$fetch` 调用，SSR 那一趟在服务端执行、水合后那一趟在浏览器执行。地址解析在两端的差异，就是 baseURL 类 bug 的来源。这一篇把这条一致性讲透。

## 为什么两端会不一样

先看两端的机制差：

- **服务端（SSR 阶段）**：`$fetch('/api/xxx')` 是**直调那个 handler 函数**，不发网络请求（300、318 那条"省一次 API 调用"）；但它也**不会自动带这次请求的 cookie/authorization 头**（"due to security considerations"，318），要带得用 `useRequestFetch()` 或 `useFetch`（后者底层就是它）。
- **浏览器**：`$fetch('/api/xxx')` 是真发 HTTP 请求，浏览器自动带上 cookie 等。

两端都是"相对 `/api`"就还好；一旦掺进绝对 URL 或 baseURL，差异就出来了。getting-started/server 那页也提醒 handler 能"直接 return text/json/html/stream"，SSR 直调拿到的是函数返回值、浏览器拿到的是 HTTP 响应——两者内容应一致，前提是你地址写对。

## 绝对 URL vs 相对路径

- 相对 `/api/...`：两端都指向"你这个应用"。SSR 直调自家 handler，浏览器打自家 `/api`。一致。
- 绝对 `http://some-api.com/...`：SSR 阶段服务端真打那个外部域名（且不带浏览器 cookie），浏览器也打它。这没"一致性 bug"，但要清楚服务端那一趟是真实外部网络调用、还得自己带需要的头。

典型翻车：在 `useFetch` 里写了个 baseURL 或绝对地址，结果 SSR 和浏览器解析出的 host 不同（比如 SSR 用 `http://localhost` 打自己、浏览器用真实域名）。314/316 讲 host 时那条"缓存命中 host 变 localhost"也是同源问题——host/origin 的解析在两端不一定一致。

## baseURL 的三层（复述 245 的口径）

要给取数设 baseURL，官方给的三层，从"最局部"到"全局"：

1. **请求选项 `baseURL`**：单次调用传（`useFetch(url, { baseURL })`）。
2. **`runtimeConfig.public`**：放一个公开基址（如 `apiBase`），能被 `NUXT_PUBLIC_API_BASE` 运行时覆盖（265）。
3. **`app` 命名空间**：`app.baseURL`（默认 `/`）、`cdnURL` 等，是 Nuxt 应用自身挂载路径，保留命名空间、别自加键。

`useFetch` 的 `baseURL` 选项是"这次请求"的前缀；`app.baseURL` 是"应用挂在子路径下时"的全局前缀。两者别混用成同一个东西。

## 一处文档没写清、必须实测

诚实标一条：当应用部署在**子路径**（`app.baseURL` 不是 `/`）时，一个相对 `useFetch('/api/...')` 到底会不会自动带上 `app.baseURL` 前缀——**4.x 文档没有把这条写成规则**。我在 245/222 就标过这个空白。所以别猜，用这个方法验：

1. 设一个非 `/` 的 `app.baseURL`（如 `/my-app/`）。
2. `useFetch('/api/x')`。
3. 打开浏览器 Network 看这条请求打的是 `/api/x` 还是 `/my-app/api/x`；SSR 端在日志里看直调命中的路径。

两端都打中你要的那个、且一致，才算这个子路径部署没问题。

## 一致性排查判据

遇到"SSR 取到 A、水合后取到 B / 报错"：

- 先看 URL 是绝对还是相对、有没有 baseURL，且它在哪一层设（选项 / public / app）。
- 服务端那一趟要不要带登录态头 → 用 useFetch 或 useRequestFetch（315/318）显式转发。
- 子路径部署 → 用上面那个 Network 实测法验相对路径带不带 app.baseURL，别信"应该会"。
- host 相关（SSR 拿到 localhost）→ 混合渲染命中缓存丢头，配 cache.varies（316）。

## 自查

- 优先用相对 `/api` 路径，让两端都指向自家应用；服务端取数要带头用 useFetch/useRequestFetch。
- baseURL 分三层：单次请求选项、runtimeConfig.public（可 NUXT_ 覆盖）、app.baseURL（应用挂载路径，别自加键）。
- 子路径部署下相对路径带不带 app.baseURL，文档未写成规则 → 用 Network + SSR 日志实测，别猜。
- SSR 直调 handler、浏览器发 HTTP，两端内容要一致前提是地址解析对齐。

## 下一篇

下一篇 333：跨域与请求头：文档里能处理跨域的地方
