---
url: /posts/e015ac1cb712c9c6d0015b23de9a509a/
title: 039 - nuxt preview 与 nuxt start 的选项全表，以及 dev 有而它没有的那些
date: 2026-10-25T09:00:00+08:00
updated: 2026-10-25T09:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt preview（官方说 nuxt start 就是同一个命令换个名字）的全部开关只有九个：--cwd、--logLevel、--envName、-e/--extends、-p/--port（默认 NUXT_PORT 或 NITRO_PORT 或 PORT）、-h/--host（默认 NUXT_HOST 或 NITRO_HOST 或 HOST）、--takeover 与 --no-takeover、--strictPort（默认 false）、--dotenv（可重复、靠后的文件优先）。和 034 那张 dev 选项表并排看，会发现 dev 的 --open、--open.url、--clipboard、--qr、--tunnel、--public、--publicURL、整排 --https.*、--inspect 与 --inspect-brk、--tui 与 --clear、--fork 在 preview 里一个都没有——它不打开浏览器、不帮你起 TLS、不提供交互式面板，因为它跑的是构建产物而不是开发服务器。两者的"发现自己已经在跑"机制也不同：dev 把锁写在 build 目录里的 nuxt.lock，preview 把自己记录在 node_modules/.cache/nuxt/preview。

categories:
  - 前端开发

tags:
  - nuxt4
  - preview
  - cli
  - 选项表
  - 命令行

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

029 我们讲的是心智上的坑，preview 会读 `.env`、会设 `NODE_ENV=production`、端口被占会自己换。那一篇的目的是别被它骗。

这一篇是它的参考页，这条命令到底有哪些开关，以及它和 `nuxt dev` 差在哪一排选项上。后者比听起来有用，那排缺失的开关，正好告诉你 preview 不是干什么用的。

全部用法。

```bash [Terminal]
npx nuxt preview [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--envName=<environment>] [-e, --extends=<layer-name>...] [-p, --port=<port>] [-h, --host=<host>] [--takeover] [--strictPort] [--dotenv=<path>...]
```

029 提过，`nuxt start` 是同一条命令的另一个名字。

选项全表。`ROOTDIR` 默认 `.`，是你 Nuxt 项目的根目录。`--cwd=<directory>` 指定 Nuxt 项目的根目录。`--logLevel` 指定构建期日志级别。`--envName=<environment>` 是解析配置覆盖时使用的环境，构建时默认 `production`，跑开发服务器时默认 `development`。`-e, --extends=<layer-name>...` 从某个 Nuxt 图层继承。`-p, --port=<port>` 是监听的端口，默认 `NUXT_PORT || NITRO_PORT || PORT`。`-h, --host=<host>` 是监听的主机，默认 `NUXT_HOST || NITRO_HOST || HOST`。`--takeover` 与 `--no-takeover` 是停掉本项目已在运行的 preview 并取代它，或者绝不停掉它。`--strictPort` 默认 `false`，请求的端口不可用时退出，而不是换一个。`--dotenv=<path>...` 是要加载的 `.env` 路径，相对根目录，可重复，靠后的文件优先。

对照 034 的 dev 表，注意两处细微但重要的差别。

`--port` 的默认链少一环。dev 是 `NUXT_PORT || NITRO_PORT || PORT || nuxtOptions.devServer.port`，preview 到 `PORT` 为止，因为它没有 devServer 那份配置可回退，`devServer.port` 是开发服务器的默认值，见 085。

`--envName` 的默认值这一行在两页里写的是同一句话，构建时默认 `production`，跑开发服务器时默认 `development`。这不是笔误，而是在提醒，同一个 `--envName` 说明文本被复用，具体取哪个取决于你在跑哪条命令。preview 属于构建和生产那一侧，029 看过它把 NODE_ENV 设为 `production`。

dev 有、preview 没有的那一排。把 034 的 dev 选项和这张表并排，preview 里完全不存在的开关是，`-o/--open`、`--open.url`、`--clipboard`、`--qr`、`--tunnel`、`--public`、`--publicURL`、`--https` 与那六个 `--https.*` 子参数、`--inspect`、`--inspect-brk`、`--tui` 与 `--no-tui`、`--clear`、`-f/--fork` 与 `--no-fork`，以及 `--prerender`，那是 build 的。

这一排缺失，翻译过来是三句话。

一，preview 不替你打开浏览器、不给你二维码、不建隧道。它是我要看这份构建产物跑起来对不对，不是我要把本地界面分享出去。要那些能力，回到 dev。

二，preview 没有 `--https`。官方在 preview 的选项表里没列任何证书参数。所以别指望用命令行让 preview 以 TLS 跑，034 那一整排 HTTPS 开关只在 dev 页出现。

三，preview 没有 `--inspect`。调试的入口在 dev 那一侧，9229 与 9230 那两个端口。这一点很关键，要调试服务端渲染逻辑，应该在 dev 上做，而不是在 preview 上想办法。preview 的价值恰恰是环境尽量接近生产，一旦你往它里面塞调试器，那个价值就打折了。

这三条是我按选项表里不存在给出的读法，官方没有单独写一节解释为什么 preview 不提供这些开关。

发现已经有一个在跑，两者用的是两套记录。033 与 034 看过 dev 的做法，在 build 目录里写 `nuxt.lock`，第二个 dev 因此报告已有的实例而不是抢端口。

preview 的做法在 029 抄过，但值得和 dev 并排放一次，因为位置不同。preview 把自己记录在 `node_modules/.cache/nuxt/preview` 里，所以同一个项目的第二个 `nuxt preview` 会按 `nuxt dev` 处理正在运行的 dev 服务器那样的方式来处理已经运行的那一个。

对照一下，dev 的记录位置是 build 目录里的 `nuxt.lock`，抢占开关是 `--takeover`，另有 `NUXT_IGNORE_LOCK=1`，官方标 unsupported。preview 的记录位置是 `node_modules/.cache/nuxt/preview`，抢占开关是 `--takeover` 与 `--no-takeover`。

那四行谁在终端里起的乘谁新起的处理表，029 已经逐条抄过，这里不再重复。要记住的是这张表只属于 preview，dev 那一侧的对应规则是报告已有实例，不是问你接管不接管。

还有一个只在 preview 页出现的细节，当另一个服务器，包括 `nuxt dev`，正使用请求的端口时，preview 会自动使用另一个空闲端口，而不会停止那个服务器。

dev 与 preview 可以在同一台机器上并存，dev 用锁保护自己是不抢端口，preview 则换端口。这就是 029 那句你在 3000 上看到的可能是 dev 而不是构建产物的机制来源。

而官方给 preview 的自动换端口划了一个范围，端口选择适用于静态 preview 以及直接用 Node.js、Bun 或 Deno 运行的 preview 命令，其它 preview 命令自行选择端口。

这条意味着，某些 preset 下的 preview，`--strictPort` 那套未必管得住它。看到它输出的端口号，以它报的为准，官方在上一段也说了命令会告诉你它正在运行什么。

两种没有服务器可跑的情况。029 抄过原文，这里只作为参考页的收尾索引。

构建本身不产出服务端，例如 client-only 构建，直接提供静态产物，未匹配路径回退到客户端入口。

preset 不产可在本地运行的服务器，改为运行该 preset 自己的 preview 命令，并打印它在跑什么。

一个和 040 的联动。preview 的记录文件躺在 `node_modules/.cache/nuxt/preview`，而 040 会看到 `nuxt cleanup` 的清单里就有 `node_modules/.cache` 这一项。

所以，跑过一次 cleanup，那个已经有一个 preview 在跑的状态就没了记录。如果你之后遇到明明有进程还在、命令却说没有，这一条能解释，它不是端口的问题，是记录文件被清掉了。这个因果关系来自两页清单的重合，官方没有单独写这条提醒。

## 自查

- [ ] preview 的 `--port` 默认链和 dev 差在哪一环？为什么？
- [ ] 想在 preview 上开 TLS 或调试器，官方给了开关吗？该回到哪条命令？
- [ ] dev 与 preview 各自把「我已经在跑」记在哪里？两者能并存吗？
- [ ] 哪些 preview 不适用官方那条「自动换空闲端口」的规则？
- [ ] 跑过 `nuxt cleanup` 之后，preview 的接管判断会受什么影响？

## 下一篇

040 讲 `nuxt cleanup`：它删掉的正好是五处生成物与缓存，其中两处（`.nuxt`、`node_modules/.cache`）一删，就把你 035、036 辛苦备好的类型与「已运行的 preview」状态一起带走了。
