---
url: /posts/4b20f8bcc0c1d86a9d40b5287a137aeb/
title: 475 - 浏览器与 Node 双端调试的官方做法
date: 2027-03-19T10:00:00+08:00
updated: 2027-03-19T10:00:00+08:00
author: cmdragon

summary:
  摘要：`nuxt.config`、模块和构建钩子跑在 **CLI 进程**里，服务端路由、中间件和服务端渲染跑在 **Nitro 的 worker 线程**里——这两个进程各有各的 inspector，默认端口是 9229 与 9230。`--inspect` 开的是服务端代码那一个，默认 `127.0.0.1:9229`，Chrome DevTools（`chrome://inspect`）与多数编辑器不用配置就能找到它，服务端重载后调试器会**自动重连**；CLI 进程那个在下一个端口（默认 9230），要手动在 `chrome://inspect` 的 Configure 里加 `localhost:9230`。要在 Nuxt 加载期间就断住，用 `--inspect-brk`：CLI 会等调试器 attach 才加载 Nuxt。另有一条容易撞上的副作用：**挂了调试器时，终端的交互面板会退化成纯日志流**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 调试
  - inspector
  - Chrome DevTools
  - 性能分析
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

你在 `nuxt.config.ts` 里下断点，调试器连着，断点却不生效。改成在服务端路由里下断点，一下就中。

这不是编辑器的问题——**那两段代码根本不在同一个进程里**。官方在 `nuxt dev` 那一节的 Debugging 小节把这条分界写得很清楚。

## 两个进程，两个端口

先看官方给的这一句，它是这一篇的根：

> `--inspect` opens the Node.js inspector for your server code: server routes, middleware and server-side rendering, which run in a **Nitro worker thread**. It accepts an optional `[host:]port` and defaults to `127.0.0.1:9229`, so Chrome DevTools (`chrome://inspect`) and most editors find it without configuration. Your debugger reconnects automatically when the server reloads.

四个信息：**能断的对象是服务端路由、中间件、服务端渲染**；它们跑在 Nitro 的 worker 线程；默认地址端口 `127.0.0.1:9229`；**服务端重载后调试器自动重连**（这条让"改一次代码就要重新 attach"这种事不再发生）。

然后是 CLI 那一侧：

> `nuxt.config`, modules and build hooks run in the CLI process instead, which gets its own inspector on the next port (`9230` by default). Add `localhost:9230` as a target in `chrome://inspect` (**Configure...**) or point your editor at it.

对照表：

| 代码位置 | 在哪个进程 | inspector | 怎么找到 |
| --- | --- | --- | --- |
| 服务端路由、服务端中间件、SSR | Nitro worker 线程 | 默认 `127.0.0.1:9229` | Chrome DevTools 与多数编辑器**免配置**发现 |
| `nuxt.config`、模块、构建钩子 | CLI 进程 | 默认下一个端口 `9230` | 手动在 `chrome://inspect` 的 **Configure** 里加 `localhost:9230`，或让编辑器指向它 |

开头那个现象就在这里：**默认那个 inspector 覆盖不到 CLI 进程**，所以配置里的断点不中。要断配置/模块/构建钩子，得自己把 9230 加进去。

两个开关的官方说明行：

| 选项 | 官方说明 |
| --- | --- |
| `--inspect` | 为服务端代码启用 Node.js inspector（`--inspect=[host:]port`），并在**下一个端口**为 CLI 进程也启用 |
| `--inspect-brk` | 与 `--inspect` 相同，并且**在加载 Nuxt 之前等待调试器 attach 到 CLI 进程**（`--inspect-brk=[host:]port`） |

选项行里那句"and for the CLI process on the next port"补上了一个重要事实：**`--inspect` 其实两端都开**，只是第二个目标要你自己去加。

## brk 是给"加载期间就跑"的代码

官方给 `--inspect-brk` 的用例一句话：

> To debug code that runs while Nuxt is loading, use `--inspect-brk`. The CLI process then waits for a debugger to attach to the next port before loading Nuxt:

```bash
npx nuxt dev --inspect-brk
```

判据很简单：**要看的是 Nuxt 启动过程中就执行完的代码**（配置求值、模块 `setup`、构建钩子早期阶段），普通 `--inspect` 来不及拦住它，`brk` 会让进程停在 attach 之前。

## 浏览器那一侧

客户端调试走的是 IDE 配置里的第二个目标。474 记的两份样例里，两边都有这一项：

- VS Code：`type: "chrome"`、`url: "http://localhost:3000"`、`webRoot` 注释写明**要指向你的 `srcDir`（默认 `app`）**；想沿用日常浏览器的扩展就加 `"userDataDir": false`。
- JetBrains：`type="JavascriptDebugType"`、`uri="http://localhost:3000"`、带 `useFirstLineBreakpoints="true"`。

两端一起起靠的是 compound：`fullstack: nuxt` 把 `server: nuxt` 与 `client: chrome` 组合成一个启动项。**这就是官方给的"双端调试"做法：不是某个开关，而是一份把两个目标并起来的配置。**

## 一个副作用：调试器会关掉交互面板

`nuxt dev` 那块固定面板（URL、启动进度、快捷键）的退化条件官方列了四种，其中一种是调试器：

> It falls back to a plain stream of logs when the output is not a terminal, in CI, **when a debugger is attached**, or when the terminal is too small.

所以"一 attach 调试器，终端面板就不见了"是设计行为，不是崩了。plain 模式仍然有一小组快捷键：`r` 重启、`o` 打开、`u` 显示 URL、`qr` 出二维码、`copy` 复制地址、`c` 清屏、`h` 帮助、`q` 退出。

要强行开回面板，官方给的变量是 `NUXT_TUI=1`——但那句括号里写明**输出被管道或重定向时这条不起作用**。

## 不用 attach 也能定位耗时：请求追踪

双端调试之外，官方还有一条更轻的路——dev 的网络视图（按 `n`）里选中一个请求按 `enter`：

> The trace draws everything the server did for that request on one timeline: route middleware, server route handlers, Nuxt plugins, hooks, data fetching, rendering, and requests your app made to other servers. Below the timeline, it lists how long Vite spent compiling modules for the request, broken down by Vite plugin and slowest module, followed by the logs the request produced.

清单里的对象已经把"服务端一次请求做过的事"全列了：路由中间件、服务端路由处理器、Nuxt 插件、钩子、取数、渲染，以及你的应用向其他服务器发的请求。下面还有 Vite 编译该请求所用模块的耗时，**按插件和最慢模块拆开**，最后是这条请求产生的日志。

它的来源官方也写了：**这些 trace 由 Nuxt 与 Nitro 发布的 tracing channels 构建**，所以显示多少取决于版本；`nuxt dev` 会替你把 `tracingChannel` 选项打开，**不想要就在 `nuxt.config` 里设 `tracingChannel: false`**。

## 性能这一路：--profile

命令侧的选项行写的是"Profile performance, writing a V8 CPU profile and a JSON report on exit. Use `--profile=verbose` for a full console report."，细节节补了文件名与时机：

> `--profile` writes a V8 CPU profile to `nuxt-dev.cpuprofile` in your project when the process exits. `--profile=verbose` also prints a full report to the console.

还有一条版本限制值得先查，免得找不到文件：

> The build timings, `perf-report.json` and `perf-trace.json` come from Nuxt's own build profiling, which needs Nuxt v4.4 or later. On an earlier version the CPU profile is still written.

也就是：**构建计时与那两个 JSON 报告要求 Nuxt v4.4 及以上**；低版本只写那个 CPU profile 文件。

## 自查

- 两端不是同一进程：服务端代码在 Nitro worker 线程（默认 9229，免配置发现、重载自动重连），配置/模块/构建钩子在 CLI 进程（默认 9230，需手动加 attach 目标）。
- `--inspect` 两个都开，第二个要自己接；要看 Nuxt 加载期间的代码用 `--inspect-brk`。
- 浏览器侧靠 IDE 的 `client: chrome`（`webRoot` 跟 `srcDir` 走），双端一起起用 compound 配置。
- attach 调试器会让 TUI 退化成纯日志流，这是设计行为。
- 只想看一次请求花在哪，用网络视图的请求追踪（tracing channels，可用 `tracingChannel: false` 关掉）；CPU 侧用 `--profile`，构建计时那两个报告要 v4.4+。

## 下一篇

下一篇 476 讲报障这件事：官方报障页的四步是"先分清是求助还是 bug、先搜 open issues 与 discussions、做最小复现、判断责任在哪一层"，其中两条有硬性指引——**能归到某个模块或上游库就别提到 Nuxt 仓库**，以及**问题若涉及 Vue 或 Vite，先用 Vue SSR starter 试着复现**。
