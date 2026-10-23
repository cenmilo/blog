---
url: /posts/69ba9f0e3b9e7f02851ffd38ad66c77d/
title: 511 - build 与 preview 的闭环：两条命令的参数表差异
date: 2027-03-31T10:00:00+08:00
updated: 2027-03-31T10:00:00+08:00
author: cmdragon

summary:
  摘要：`nuxt build` 创建 `.output` 目录，把应用、服务器和依赖都准备成生产可用，并报告构建耗时；`nuxt preview` 在 build 之后起一个服务器来预览它，`nuxt start` 是同一件事的另一个名字。两条命令的选项表有重叠但各不相同：`build` 独有 `--prerender`（会把 target 设成 `static`）、`--target`（`--preset` 是它的别名）与 `--profile`；`preview` 独有 `-p/--port`、`-h/--host`、`--takeover`/`--no-takeover` 与 `--strictPort`。两条都会把 `NODE_ENV` 设为 `production`，但 preview 会替你加载 `.env`，而生产环境不会。

categories:
  - 前端开发

tags:
  - nuxt4
  - cli
  - 构建
  - 预览
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`nuxt build` 成功了，接下来怎么在本地确认这个产物真能跑？把 `nuxt dev` 再开一遍是不够的——dev 跑的是开发流程，不是你的产物。

官方给这一环的命令是 `nuxt preview`。两条命令是一对，选项表却不是一份。

## 两条命令各说什么

`build` 的定性：**"The `build` command creates a `.output` directory with all your application, server and dependencies ready for production, and reports how long the build took."**

`preview` 的定性里多给了一个别名和一句转介：

> The `preview` command starts a server to preview your Nuxt application after running the `build` command. `nuxt start` is the same command under another name. When running your application in production refer to the Deployment section.

也就是说：**预览用的服务器与生产启动用的服务器，在官方口径里是两件事**——预览归这条命令，生产归部署那一节的 `node .output/server/index.mjs`。

命令形状（原文照抄）：

```bash [Terminal]
npx nuxt build [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--prerender] [--target=<target>] [--dotenv=<path>...] [--envName=<environment>] [-e, --extends=<layer-name>...] [--profile=<verbose>]
```

```bash [Terminal]
npx nuxt preview [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--envName=<environment>] [-e, --extends=<layer-name>...] [-p, --port=<port>] [-h, --host=<host>] [--takeover] [--strictPort] [--dotenv=<path>...]
```

## 选项表：谁有什么

两条命令的位置参数都只有一个 `ROOTDIR`（默认 `.`），有五个选项完全共用：`--cwd`、`--logLevel`（都是 **build-time** log level）、`--envName`、`-e/--extends`、`--dotenv`（可重复，**后面的文件优先**）。

| 只在一侧出现 | 官方说明 |
| --- | --- |
| `build` 的 `--prerender` | Build Nuxt and prerender static routes；另有一条 `::note`：**`--prerender` will always set the target to `static`** |
| `build` 的 `--target=<target>` | Deploy target for the configured server builder（示例给了 `node-server`、`vercel`、`netlify`） |
| `build` 的 `--profile=<verbose>` | 性能剖析，退出时写 V8 CPU profile 与 JSON 报告；`--profile=verbose` 才打完整控制台报告 |
| `preview` 的 `-p, --port=<port>` | 监听端口，默认取 `NUXT_PORT \|\| NITRO_PORT \|\| PORT` |
| `preview` 的 `-h, --host=<host>` | 监听主机，默认取 `NUXT_HOST \|\| NITRO_HOST \|\| HOST` |
| `preview` 的 `--takeover` / `--no-takeover` | 总是停掉本项目已在跑的预览服务并接管 / 绝不停掉 |
| `preview` 的 `--strictPort` | 默认 `false`；设了之后端口不可用就直接退出，而不是换一个 |

`--envName` 那一条的说明在两个页面上逐字相同，值得单独记：**"default is `production` when building, and `development` when running the dev server"**——它没说 preview，但 preview 是构建后的运行，环境覆盖按 `production` 解析（这一点是 `--envName` 定义本身加部署页 `NODE_ENV` 表述能支持的读法，页面上没有直接写 preview 用哪个环境）。

## build 侧的三个补充事实

**`--preset` 还在，但只是别名。** 原文：`--preset` is still accepted as another name for `--target`. When neither is passed, the `NITRO_PRESET` and `SERVER_PRESET` environment variables are used, in that order. 三者优先级顺序在这句里给全了：命令行两个名字等价，都不给才看环境变量，且 `NITRO_PRESET` 先于 `SERVER_PRESET`。

**`--prerender` 会改 target。** 那条 `::note` 是硬结果：target 变成 `static`，不是你原来给的那个。

**`--profile` 的产物与版本要求。** 写到项目里的 `nuxt-build.cpuprofile`；构建计时连同构建目录里的 `perf-report.json` 与 `perf-trace.json` 来自 Nuxt 自己的构建剖析，**需要 Nuxt v4.4 或更高**。

## preview 侧最实用的两段：它跑的是什么、端口被占了怎么办

第一段是两种"没有服务器可跑"的情况，官方都写了替代行为：

- 配置里的 `server.builder` 根本没产出服务器时（官方举例 client-only build）：**直接静态服务那份产物**，未匹配路径回退到客户端入口，好让客户端路由还能用。
- 某些 Nitro 预设不产出能在本地跑的服务器：改为**运行该预设自己的 preview 命令**，并且命令会告诉你它正在跑什么。

第二段是接管规则。预览把自己记录在 `node_modules/.cache/nuxt/preview`，于是同一个项目第二次 `nuxt preview` 会按下表处理（四种组合，官方表格原样）：

| 已跑的预览是这样启动的 | 新的预览是这样启动的 | 结果 |
| --- | --- | --- |
| 无终端 | 无终端 | 停掉旧的，新的接管它的端口 |
| 无终端 | 有终端 | 询问你，默认接管 |
| 有终端 | 无终端 | 新的退出，并告诉你旧的在哪 |
| 有终端 | 有终端 | 询问你，默认不启动 |

`--takeover` 表示总是接管，`--no-takeover` 表示绝不。选了 "Start anyway" 或者传了不同 `--port`，就是**并排跑第二个**。

端口被别的服务器（官方明确包括 `nuxt dev`）占用时，preview 会**自动换一个空闲端口**而不停掉那个服务；要它直接失败就传 `--strictPort`。请求的端口来自 `--port`、`NUXT_PORT`、`NITRO_PORT` 或 `PORT`，默认 `3000`。这套端口选择**只适用于静态预览和直接用 Node.js／Bun／Deno 跑的预览**，其他预览命令自己决定端口。

## 最容易出事的一处差异：`.env`

那条 `::note` 写得很直白：

> For convenience, in preview mode, your `.env` file will be loaded into `process.env`. (However, in production you will need to ensure your environment variables are set yourself. For example, with Node.js 20+ you could do this by running `NODE_ENV=production node --env-file .env .output/server/index.mjs` to start your server.)

这一条能解释一整类"预览一切正常、上线白屏"：**preview 帮你读了 `.env`，生产环境没人读**。官方给的补法是用 Node 20+ 的 `--env-file`，或者由托管平台注入变量。

至于 `NODE_ENV`，两条命令都会把它设成 `production`；`preview` 那一节还补了覆盖方式——在 `.env` 里定义 `NODE_ENV`，或者作为命令行参数传。

## 自查

- `build` 造 `.output` 并报耗时；`preview` 在 build 之后起服务器预览，`nuxt start` 是同义名字，生产启动归部署页。
- 五个选项共用，`--prerender`/`--target`/`--profile` 只在 build，端口与接管相关四项只在 preview。
- `--prerender` 一定把 target 变成 `static`；`--preset` 只是 `--target` 的别名，都不给才按 `NITRO_PRESET` → `SERVER_PRESET` 顺序读环境变量。
- 没有服务器可跑时 preview 有两条后备：静态直供＋回退客户端入口，或调用预设自己的 preview 命令。
- 端口被占默认自动换（`--strictPort` 才失败），且这套规则只管静态预览与 Node/Bun/Deno 直跑。
- **preview 会读 `.env`，生产不会**——环境变量必须在生产侧自己设。

## 下一篇

下一篇 512 讲 `.output` 这个目录：官方那一节只有四句话，三条硬警告；目录内部到底有什么，得从部署页、预渲染页和 ESM 那一节拼出来。
