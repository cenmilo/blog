---
url: /posts/a340547181d0e4bfa66cfc688c67ea90/
title: 715 - CLI 内幕：nuxt.lock、进程与端口占用
date: 2027-06-07T10:00:00+08:00
updated: 2027-06-07T10:00:00+08:00
author: cmdragon

summary:
  摘要：`nuxt dev` 跑起来之后其实有三个执行位置：**CLI 进程**里跑 `nuxt.config`、模块与构建钩子；**Nitro 工作线程**里跑服务端路由、中间件与服务端渲染；浏览器里跑客户端。这就是为什么调试要开**两个 inspector 端口**——默认 `127.0.0.1:9229` 给服务端代码，下一个端口 `9230` 给 CLI 进程。进程之间还有一把锁：开发服务器会把自己记录在项目构建目录（默认 `.nuxt`）里的 `nuxt.lock` 文件中，第二个 `nuxt dev` 因此是报告"已经有一个在跑"，而不是去抢端口。

categories:
  - 前端开发

tags:
  - nuxt4
  - 命令行
  - 开发服务器
  - 进程模型
  - 调试
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

命令页上那三十三行选项与一个参数的逐项拆解，讲 `nuxt dev` 那条命令的那篇已经落地过；这一篇只讲它背后的进程与资源模型。

## 三个执行位置，对应两套调试端口

`dev` 那一节把这件事写在 Debugging 一节的开头：`--inspect` 打开的是**你的服务端代码**的 Node.js inspector——服务端路由、中间件、服务端渲染，**它们跑在一个 Nitro 工作线程里**。它接受可选的 `[host:]port`，**默认 `127.0.0.1:9229`**，所以 Chrome DevTools（`chrome://inspect`）与多数编辑器不用配置就能找到；服务端重载时你的调试器会自动重连。

而另外那部分代码在**另一个进程**：

> `nuxt.config`、模块与构建钩子跑在 CLI 进程里，这个进程有它自己的 inspector，在下一个端口（默认 `9230`）。

要挂上去得手动加目标：在 `chrome://inspect` 里用 Configure 把 `localhost:9230` 加成一个 target，或者让你的编辑器指过去。

还要调试"Nuxt 加载过程中"的代码时用 `--inspect-brk`：**CLI 进程会在加载 Nuxt 之前，等调试器附加到下一个端口**。

`--profile` 的产物位置也写清了：进程退出时把 V8 CPU profile 写到项目里的 `nuxt-dev.cpuprofile`，`--profile=verbose` 还会把完整报告打到控制台。旁边那条 note 是一个版本条件：**构建耗时、`perf-report.json` 与 `perf-trace.json` 来自 Nuxt 自己的构建剖析，需要 Nuxt v4.4 或更新；低版本仍然会写那个 CPU profile。**

## `nuxt.lock`：为什么第二次启动不报端口冲突

这一节的原文：

> 开发服务器会把自己记录在项目构建目录里的 `nuxt.lock` 文件中，因此同一个项目的第二个 `nuxt dev` 会报告那个已经在运行的实例，而不是与它抢端口。

三种应对：

- `--takeover`：**停掉这个项目上已在运行的开发服务器，取代它启动**。
- `NUXT_IGNORE_LOCK=1`：仍然起第二个服务器，文档对这个选项的态度是 **unsupported**。
- `--no-takeover`：**绝不停掉**这个项目上已在运行的开发服务器。

这把锁还服务别的命令：**`nuxt curl` 与 `nuxt task` 就是靠同一个锁文件找到要对话的服务器的**，curl 那一节也重复了这一点并补了一句：因此这些命令只在 `nuxt dev` 正在为该项目运行时有效；传绝对 URL 才能去跟别的服务对话。

构建目录默认是 `.nuxt`，但如果配置里的 `buildDir` 改了，锁文件跟着去别处。

## 端口与 host 的解析顺序，以及 `--strictPort`

页面上这条链写了两次（选项表一次、环境变量表一次），顺序是一样的：端口依次取 `NUXT_PORT`、`NITRO_PORT`、`PORT`，都没有就用配置里的 `nuxtOptions.devServer.port`；host 依次取 `NUXT_HOST`、`NITRO_HOST`、`HOST`，都没有用 `nuxtOptions.devServer?.host`。

**默认行为是端口被占用时换一个**，因为 `--strictPort` 的默认是 `false`；把它设上，请求的端口不可用时就**直接退出而不是换一个**。这个差别对 CI 与脚本很重要：默认行为会让"端口被别的进程占了"表现成"服务起在了另一个端口上"。

另外这一节还写了一句容易被忽略的事实：**这条命令会把 `process.env.NODE_ENV` 设成 `development`。**

## reload 与 restart 不是一回事

Restarts 那一节的区分很干脆：

> **reload 是原地进行的；restart 会替换进程**——改 `nuxt.config` 或改一个已安装的依赖需要的就是 restart。

而且它说了可见性：服务器重载或重启时会说明是什么引起的，**改 `nuxt.config` 引起的重启还会列出实际有差异的键**。所以一次只是格式化过的编辑，与一次真的改了配置的编辑，在输出上是能分辨的。

## fork：应用由子进程提供服务

`-f, --fork` 的默认在选项表里写的是 **runtime-dependent**，说明句是"从一个被 fork 出来的子进程里为应用提供服务（**在运行时支持的地方默认开启**）"，对应的关闭项是 `--no-fork`。

这一条与前面那套进程模型是同一件事的两面：主进程管构建与配置，服务与渲染在别处（子进程、Nitro worker thread）跑。

## TUI 的退回条件与两套快捷键

`--tui` 默认 `true`：在交互式终端里渲染一个固定的面板——服务器 URL、启动进度、当前状态、一排快捷键，日志折叠在它上方。**当输出不是终端、在 CI 里、附加了调试器、或者终端太小，它会退回到纯日志流。**

交互面板的快捷键十二个：`r` 重启、`shift-r` 清缓存后重启、`o` 在浏览器打开、`y` 复制服务器 URL、`i` 显示版本/URL/二维码/会话信息、`l` 浏览日志历史、`e` 打开日志并定位到最后一次错误、`n` 浏览被服务的请求、`p` 浏览页面与服务端路由、`c` 清日志清请求清控制台、`?` 显示全部快捷键、`q` 退出。

视图内还有两个：`y` 复制选中行，`shift-y` 复制过滤与搜索剩下的所有行（**内容太多放不下时保留最新的**）；在 info 视图里 `shift-y` 复制的是 `nuxt info` 那张表。

要关掉面板：`--no-tui`，或者设 `NUXT_TUI=plain`。**也可以反过来强制开启：`NUXT_TUI=1`——但页面注明输出被管道或重定向时这一条不起作用。** 纯输出的快捷键是一组更小的集合：`r`、`o`、`u`（显示 URL）、`qr`、`copy`、`c`、`h`、`q`。

## 请求追踪是怎么拼出来的

在 network 视图（`n`）里选中一条请求按 `enter`，就能看到它的时间都花在哪。页面描述这张时间线包含的内容：路由中间件、服务端路由 handler、Nuxt 插件、钩子、数据获取、渲染，以及**你的应用向其他服务器发出的请求**。时间线下面列出 Vite 为这条请求编译模块花了多久，**按 Vite 插件与最慢模块拆开**，再往下是这条请求产生的日志。

机制说明在最后：这张 trace 由 Nuxt 与 Nitro 发布的 tracing channels 构建，**所以它能显示多少取决于你的版本**。`nuxt dev` 会替你打开 `tracingChannel` 这个选项；不想要就在 `nuxt.config` 里设 `tracingChannel: false` 退出。

## HTTPS 的两个层次

`--https` 用 TLS 提供服务：**装了 `mkcert` 就生成一张本地受信任的证书，否则生成自签证书**。已有证书用 `--https.cert` 与 `--https.key`；PKCS#12 密钥库用 `--https.pfx` 配 `--https.passphrase`。生成证书还能控两件事：`--https.validityDays=<days>` 的有效期天数、`--https.domains=<domain>...` 的域名（可重复传，也可用逗号分隔的一个列表）。

那条 note 是个真会咬人的点：

> Node 不读你的系统信任库，所以从 Node 发出的、指向使用生成证书的服务器的请求**不会信任它**。把签发它的证书机构设进 `NODE_EXTRA_CA_CERTS`。

也就是说浏览器里绿锁正常，`useFetch` 在服务端调它却报证书错误，原因就在这条。

## 环境变量七个

页尾那张表逐行落地：

| 变量 | 用途 |
| --- | --- |
| `NUXT_PORT`、`NITRO_PORT`、`PORT` | 要监听的端口，**按这个顺序取优先级** |
| `NUXT_HOST`、`NITRO_HOST`、`HOST` | 要监听的 host，同样按顺序 |
| `NUXT_TUI` | `1` 强制开启交互 UI，`plain` 退出 |
| `NUXT_TERM_THEME` | `light` 或 `dark`，用于终端背景色检测不出来时 |
| `NUXT_IGNORE_LOCK` | `1` 表示忽略这个项目上已在运行的开发服务器 |
| `NUXT_IGNORE_UPDATE_CHECK` | `1` 让 CLI 不再检查更新的 Nuxt 发布 |

最后一条对应一个你可能没注意的行为：**CLI 默认会检查是否有更新的 Nuxt 版本**。

## 清理命令删掉的是五处

`npx nuxt cleanup [ROOTDIR] [--cwd=<directory>]` 删除常见生成文件与缓存，页面列了五项：

- 构建目录（默认 `.nuxt`，`buildDir` 改了就跟它）
- `.output`
- `dist`
- `node_modules/.vite`
- `node_modules/.cache`

这份清单有两点要留意：**它不删 `node_modules` 本身**，也**不删锁文件**——依赖问题与构建缓存问题要分开处理，后者一条 cleanup 就够，前者还得动包管理器。

## 四种表现对应的检查点

- "端口对不上"：先看 `--strictPort` 有没有开，再核对那三条端口环境变量与 `devServer.port`。
- "第二个 dev 起不来"：这是 `nuxt.lock` 在正常起作用，要取代就 `--takeover`。
- "断点永不命中"：先确认代码在哪一侧——`nuxt.config` 与模块在 CLI 进程（9230），路由与 SSR 在 Nitro 工作线程（9229）。
- "浏览器能访问但服务端 fetch 报证书错"：`NODE_EXTRA_CA_CERTS` 没设。

下一篇讲 `nuxt docs` 这条命令与它的离线文档检索：它和 MCP、llms.txt 是三种不同的"从终端查文档"的路。

## 下一篇

下一篇 716：nuxt docs 命令与离线文档检索：查的是你项目那一个版本

## 本文依据

- https://nuxt.com/docs/4.x/api/commands/dev
- https://nuxt.com/docs/4.x/api/commands/cleanup
- https://nuxt.com/docs/4.x/api/commands/curl
