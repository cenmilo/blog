---
url: /posts/00389f09960c2884e292eb402c849895/
title: 029 - 构建之后本地验一遍：nuxt preview 与真正的生产启动命令
date: 2026-10-21T11:00:00+08:00
updated: 2026-10-21T11:00:00+08:00
author: cmdragon

summary:
  摘要：官方把 preview 定位为"build 之后启动一个服务器来预览你的应用"，并且明确 nuxt start 只是同一个命令换个名字，真要在生产环境运行要看 Deployment 一节。preview 有三条规则很容易让人误判：第一，它会把 process.env.NODE_ENV 设为 production；第二，出于方便它会把你的 .env 文件加载进 process.env，而生产环境没人替你加载，环境变量要自己设（官方给的例子是 NODE_ENV=production node --env-file .env .output/server/index.mjs）；第三，如果请求的端口已经被别的服务器（包括 nuxt dev）占用，它会自动换一个空闲端口继续跑，除非你传 --strictPort。真正部署到 Node 服务器时，产物入口是 .output/server/index.mjs，官方要求带 NODE_ENV=production 启动，因为有些依赖（尤其 Vue Router）只有在这个变量设了之后才会去掉开发期警告。

categories:
  - 前端开发

tags:
  - nuxt4
  - preview
  - 构建
  - 部署
  - 环境变量

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`npm run dev` 跑起来了、数据也能显示了。但**开发服务器跑通不等于生产产物跑得通**——这两者跑的东西根本不一样（012 讲过一次构建产出三样东西）。这一篇教你用最低成本验证一次生产产物，并且讲清 preview 为什么是个"会骗人"的命令。

## 命令长什么样

官方对 `preview` 的定义：

> `preview` 命令启动一个服务器，用于在跑完 `build` 命令之后预览你的 Nuxt 应用。`nuxt start` 是同一个命令换了个名字。当你在生产环境运行应用时，请参考 Deployment 一节。

三句话三个信息：它是**构建之后**用的；它和 `nuxt start` 是**同一个东西**；它**不是生产运行方式**（生产去 Deployment 那节）。

用法：

```bash [Terminal]
npx nuxt preview [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--envName=<environment>] [-e, --extends=<layer-name>...] [-p, --port=<port>] [-h, --host=<host>] [--takeover] [--strictPort] [--dotenv=<path>...]
```

## 完整流程：build 然后 preview

```bash [Terminal]
npm run build
npm run preview
```

（对照 024 那张表：`build` → `nuxt build`，`preview` → `nuxt preview`。）

`build` 的产物落在 `.output/`，preview 起的服务就是它。053 讲 `.output` 目录时会拆开看里面有什么。

## 第一条容易误判的规则：preview 会读 `.env`，生产不会

官方在 preview 页末尾的 note 是这样的：

> 为了便利，在 preview 模式下，你的 `.env` 文件会被加载进 `process.env`。（但是，在生产环境中你需要自己确保环境变量被设置好。例如，使用 Node.js 20+ 你可以这样启动服务器：`NODE_ENV=production node --env-file .env .output/server/index.mjs`。）

**这是"本地看着好好的、上线就炸"最经典的一条来源。** 回想 018 讲的 runtimeConfig 规则：运行时环境变量会自动覆盖同名配置项——但如果那个变量在生产进程里根本不存在，覆盖就无从发生，你的应用拿到的会是 `nuxt.config` 里写的默认值。

判断标准很硬：**凡是只在 `.env` 里配的运行时配置，本地 preview 通过不代表线上通过。**

## 第二条：它把 NODE_ENV 设成了 production

官方原文：

> 这个命令会把 `process.env.NODE_ENV` 设为 `production`。要覆盖它，请在 `.env` 文件里定义 `NODE_ENV`，或作为命令行参数传入。

这一条其实是**好事**：preview 至少把 NODE_ENV 摆对了。官方在 Deployment 页专门用 important 块解释了为什么这个变量重要：

> 运行服务器时请设置 `NODE_ENV=production`。某些依赖（**尤其是 Vue Router**）只有在设置了它之后才会去掉仅开发用的警告，所以不设它可能被 `[Vue Router warn]: No match found for location with path …` 这类消息淹没日志。

也就是说：**部署后日志里那一大片 Vue Router 警告，多半不是你代码错了，而是启动命令少写了 `NODE_ENV=production`。**

## 第三条：端口被占用时它会自己换

官方在 preview 页写得很细：

> 当另一个服务器（**包括 `nuxt dev`**）正使用请求的端口时，preview 会自动使用另一个空闲端口，而不会停止那个服务器。传 `--strictPort` 则改为退出。请求的端口来自 `--port`、`NUXT_PORT`、`NITRO_PORT` 或 `PORT`，默认为 `3000`。

这段有两个新手会撞上的后果：

1. **你 `nuxt dev` 还开着，又去跑 preview。** 结果 preview 静默换了端口。你在 `3000` 上看的还是 dev 那个热更新版本，却以为自己验的是构建产物。
2. **想让它别自作主张**，加 `--strictPort`（官方标的默认值是 `false`，即默认允许换端口）。

相关的其它端口规则，从官方的选项表里能直接读出：`-p, --port` 默认取 `NUXT_PORT || NITRO_PORT || PORT`，`-h, --host` 默认取 `NUXT_HOST || NITRO_HOST || HOST`。

## preview 自己怎么记着"我已经在跑"

官方给了一段机制说明：

> preview 把自己记录在 `node_modules/.cache/nuxt/preview` 里，所以同一个项目的第二个 `nuxt preview` 会按 `nuxt dev` 处理正在运行的 dev 服务器那样的方式来处理已经运行的那一个。

后面跟了一张四行表，值得存下来（行=已运行的那个是怎么起的，列=新起的这个）：

| 已运行的 preview 启动方式 | 新 preview 启动方式 | 结果 |
| --- | --- | --- |
| 无终端 | 无终端 | 旧的被停止，新的占用它的端口 |
| 无终端 | 在终端里 | 询问你，默认是接管 |
| 在终端里 | 无终端 | 新的退出，并告诉你旧的那个在哪 |
| 在终端里 | 在终端里 | 询问你，默认是不启动 |

> 传 `--takeover` 表示总是停掉旧的并在其位置启动；传 `--no-takeover` 表示从不这么做。在提示中选择"Start anyway"，或者传一个不同的 `--port`，则会在它旁边再跑一个 preview。

## 两种 preview 跑不起来的情况

官方明确列了两条，都不是错误：

**一、构建里根本没有服务端。**

> 当配置的 `server.builder` 根本不产出服务器时（例如 client-only 构建），就没有东西可运行：取而代之的是直接提供静态产物，未匹配到的路径回退到客户端入口，以便客户端路由正常工作。

**二、某些 Nitro preset 本地产不出可运行的服务器。**

> 一些 Nitro preset 不产生能在本地运行的服务器。对于这些，会改为运行该 preset 自己的 preview 命令，并且命令会告诉你它正在运行什么。

第二条 reassuring 的地方是**它会打印出来自己在跑什么**，所以你不会被蒙在鼓里；看到那行输出别当成报错。

## 真正部署时跑的是什么

Deployment 页开头给了总的分类：

> 一个 Nuxt 应用可以部署在 Node.js 服务器上、为静态托管做预渲染、或部署到 serverless 或 edge（CDN）环境。

其中 Node.js 服务器这一条是最通用的，官方给的启动命令就是这一行：

```bash [Terminal]
NODE_ENV=production node .output/server/index.mjs
```

> 这会启动你的生产 Nuxt 服务器，默认监听端口 3000。

对照官方对 Node preset 的三条描述：

> - 未指定或未自动检测到格式时的**默认输出格式**
> - 只加载渲染请求所需的 chunk，以获得最佳的冷启动时间
> - 适用于把 Nuxt 应用部署到任何 Node.js 托管

**这里就是 preview 和生产启动命令的真正分界**：两者跑的都是 `.output`，但生产环境里**没有人在替你加载 `.env`**，而且 `NODE_ENV=production` 要你自己写进启动命令。

它认这些运行时环境变量（官方列的）：

- `NITRO_PORT` 或 `PORT`（默认 `3000`）
- `NITRO_HOST` 或 `HOST`（默认 `'0.0.0.0'`）
- `NITRO_SSL_CERT` 与 `NITRO_SSL_KEY`——两者都存在时以 HTTPS 模式启动。**官方紧接着说：绝大多数情况下这只该用于测试，Nitro 服务器应当跑在像 nginx 或 Cloudflare 这样的反向代理之后，由它们终结 SSL。**

## 自查

- [ ] `nuxt preview` 和 `nuxt start` 是什么关系？官方让你在生产运行前去看哪一节？
- [ ] 你本地 preview 通过，`.env` 里的运行时配置在线上会生效吗？为什么？
- [ ] `nuxt dev` 还开着时跑 preview，会发生什么？怎么强制它别换端口？
- [ ] preview 会把 `NODE_ENV` 设成什么？不设这个变量，Vue Router 会给你看什么？
- [ ] 生产启动那一行命令里，入口文件的路径是什么？

## 下一篇

030 把 024 里那五种包管理器写法彻底对齐：`create` 与 `dev` 的写法差异从哪来、锁文件各自叫什么、`npx` 什么时候必须换成 `npm run`。

## 本文依据

- `api/commands/preview`（定义：启动服务器以在 build 之后预览应用、nuxt start 是同一命令的另一个名字、生产运行参考 Deployment 一节；完整用法命令行与 ROOTDIR/--cwd/--logLevel/--envName/-e extends/-p port/-h host/--takeover/--strictPort/--dotenv 选项，port 默认 NUXT_PORT||NITRO_PORT||PORT、host 默认 NUXT_HOST||NITRO_HOST||HOST、strictPort 默认 false；server.builder 不产出服务器时（如 client-only 构建）直接提供静态产物并以客户端入口兜底未匹配路径；部分 Nitro preset 不产出可在本地运行的服务器，改跑该 preset 自己的 preview 命令并告知在跑什么；preview 自记录在 node_modules/.cache/nuxt/preview 与四种组合的处理表、--takeover/--no-takeover、选择 Start anyway 或换 --port 则并行再起一个；端口被包括 nuxt dev 在内的其它服务器占用时自动换空闲端口且不停那个服务器，--strictPort 则退出，端口取自 --port/NUXT_PORT/NITRO_PORT/PORT 默认 3000；该命令把 process.env.NODE_ENV 设为 production，可用 .env 或命令行参数覆盖；note：preview 模式会加载 .env 进 process.env，生产需自行确保环境变量，Node.js 20+ 例子 NODE_ENV=production node --env-file .env .output/server/index.mjs）
- `getting-started/deployment`（应用可部署在 Node 服务器、为静态托管预渲染、或 serverless/edge；Node.js server preset：未指定或自动检测不到时的默认输出格式、只加载渲染请求所需 chunk 以获得最佳冷启动、适用于任何 Node 托管；entry point 命令 NODE_ENV=production node .output/server/index.mjs 并默认监听 3000；important 要求设置 NODE_ENV=production，某些依赖尤其是 Vue Router 仅在此设置后才去掉开发期警告，否则日志可能被 [Vue Router warn]: No match found for location with path … 淹没；运行时环境变量 NITRO_PORT/PORT 默认 3000、NITRO_HOST/HOST 默认 0.0.0.0、NITRO_SSL_CERT 与 NITRO_SSL_KEY 存在则以 HTTPS 启动且官方指出绝大多数情况仅用于测试、应置于 nginx 或 Cloudflare 等反向代理之后由其终结 SSL）
