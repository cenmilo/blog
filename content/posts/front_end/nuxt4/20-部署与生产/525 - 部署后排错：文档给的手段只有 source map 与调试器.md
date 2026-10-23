---
url: /posts/4659e9155a4259f9899becf1a34ff992/
title: 525 - 部署后排错：文档给的手段只有 source map 与调试器
date: 2027-04-05T09:00:00+08:00
updated: 2027-04-05T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方调试那一节只有四节：Sourcemaps、Debugging with Node Inspector、Debugging in Your IDE（VS Code 与 JetBrains 两份配置）、Other IDEs（一句欢迎开 PR）。**它没有症状对照表，也没有任何生产排错章节**。更要紧的是这节的可用范围：source map 的官方默认是"服务端构建默认开、客户端只在 dev 开"，`nuxt dev --inspect` 那句明写它是以 **dev 模式**启动调试器，而 `::important` 要求 Node 与 Chrome 跑在同一平台、并直接说**这在 Docker 里不成立**。也就是说，线上环境基本用不到这一节——生产可查的东西在日志级别、`nuxt info`、错误钩子和"能不能在 dev 复现"这几处。

categories:
  - 前端开发

tags:
  - nuxt4
  - 排错
  - 调试
  - source-map
  - 生产
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

线上出问题，你打开官方调试那一节，希望它按症状告诉你查什么——白屏查哪、404 查哪、内存涨查哪。

那一节没有这种东西。它有四节，而且每一节都是为开发环境写的。

## 这一节只有四节

整页结构与关键句：

| 小节 | 官方给了什么 | 生产能用吗 |
| --- | --- | --- |
| Sourcemaps | 默认值说明加一份配置示例 | 部分可用：它决定的是**默认有没有映射** |
| Debugging with Node Inspector | `nuxt dev --inspect`、Chrome DevTools 出现 Node 图标即可附加、一条 `::important` | **不能**：官方明写这是 dev 模式 |
| Debugging in Your IDE | VS Code 与 JetBrains 两份配置示例 | 不能（同样面向开发） |
| Other IDEs | 一句"欢迎开 PR" | — |

## source map：默认值就是那条边界

第一句是可用性的全部答案：

> Sourcemaps are enabled for your server build by default, and for the client build in dev mode, but you can enable them more specifically in your configuration.

拆成两条事实：**服务端构建默认有映射**，**客户端构建只在 dev 模式有**。所以线上浏览器里的 JS 报错是压缩后的位置，这不是配置错了，是默认值。官方给的显式写法：

```ts
export default defineNuxtConfig({
  // or sourcemap: true
  sourcemap: {
    server: true,
    client: true,
  },
})
```

两种形态都在注释里给了（`sourcemap: true` 或按 `server`/`client` 分别设）。**要不要在生产开客户端映射，官方没有给判断标准**——这一节只讲开关，没讲"开了会把什么暴露出去"，所以这一步得按你自己的安全要求定。

服务端那一侧则相反：默认就有映射，这是"线上服务端堆栈能还原到源码位置"的前提，配合下面那些日志手段才有意义。

## `--inspect` 帮不上线上

Node Inspector 那节给的是：

```bash
nuxt dev --inspect
```

紧跟的解释是 **"This will start Nuxt in `dev` mode with debugger active."**——命令本身就以开发模式启动。再下面那条 `::important` 把适用面又缩了一格：

> Note that the Node.js and Chrome processes need to be run on the same platform. This doesn't work inside of Docker.

两条合起来：**这条手段既限定 dev，又限定"非 Docker 且 Node 与 Chrome 同平台"**。你线上多半是容器、而且是生产构建，两个条件都不满足。IDE 那两节（VS Code 的一份 `launch.json`、JetBrains 的一份 `nuxt.run.xml`）同属开发期配置，同理。

也就是说，"生产进程附加调试器"这件事在 4.x 文档里**没有对应章节**。

## 线上真能查的四样

把范围收到"部署以后"，文档里可用的手段是这些：

| 手段 | 记载位置 | 能拿到什么 |
| --- | --- | --- |
| 日志级别 | `logLevel` 配置：CI 或无 TTY 时默认 `silent`（在 Vite 里译成 `silent`、webpack 里译成 `none`）；命令侧有 `--logLevel` | 决定你有没有输出可读 |
| 版本与模块快照 | `nuxt info`（`--json` 供脚本读） | 解析到的 Nuxt/Nitro/Vite、包管理器、加载的模块与 build modules |
| 客户端错误出口 | `vueApp.config.errorHandler`／`vue:error`／`app:error` 钩子 | 上报入口；注意生产错误载荷里没有 `cause` |
| CPU 剖析 | `nuxt build --profile`／`--profile=verbose`，写 `nuxt-build.cpuprofile`；构建计时与 `perf-report.json`/`perf-trace.json` 需 Nuxt v4.4+ | 构建期与命令执行期的 CPU profile |

`nuxt info` 那条在排错顺序上很值钱：报的是**解析到的**版本，能直接排除"以为是这个版本"这类误判——线上表现与本地不一致时先看它。

## 四类症状，文档各能给到哪

按症状找的话，能对应上的三条如下（都是别处记载的机制，不是这一节给的流程）：

- **打开页面白屏、点某页才报错**：新部署会让旧的带 hash 的 chunk 地址失效，官方为此提供内置硬重载，取值有四种（`automatic`、`automatic-immediate`、`false`、`manual`）。
- **硬刷新某个路径 404、点击却正常**：这是 SPA 兜底页与主机配置的问题，两条路都写在文档里（`200.html`/`404.html` 与宿主指向，以及 `experimental.serverPathFallback`）。
- **只在生产出现的水合异常**：官方点名的一类原因是 CDN 注入脚本（Cloudflare 的 Rocket Loader 与邮箱地址混淆），下一节专门讲这两项。
- **内存持续增长**：**4.x 文档没有任何内存分析、泄漏排查或堆快照的章节**；`--profile` 产出的是 CPU profile，不是堆信息。要处理这类问题，官方没有给方法，只能按 Node 平台自己的工具来做，而那已经超出这份文档的范围。

## 自查

- 官方调试页只有四节，全部面向开发环境；**没有症状对照表，也没有生产排错章节**。
- source map 的默认是"服务端开、客户端仅 dev"，线上浏览器堆栈难读来自默认值，不是配置坏了。
- `nuxt dev --inspect` 明确以 dev 模式启动，且 `::important` 说 Docker 里不成立——生产附加调试器无对应记载。
- 线上可用的是日志级别（CI 默认 silent 是第一嫌疑）、`nuxt info` 的版本快照、三个客户端/启动错误出口、以及 `--profile` 的 CPU profile。
- 白屏查 chunk 重载取值、404 查兜底页与 `serverPathFallback`、生产水合异常先查 CDN 注入；**内存增长在文档里没有方法可照抄**。

## 下一篇

下一篇 526 逐行拆 PM2 那份配置与 `node_cluster`：六个字段各管什么、`instances: 'max'` 与 round robin 的官方措辞，以及这两条路能不能同时用。
