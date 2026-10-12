---
url: /posts/c0705b160198d1c3cfbee5e017f3636f/
title: nuxt build 与 --target：构建目标怎么选
date: 2026-10-24T10:00:00+08:00
updated: 2026-10-24T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方对 nuxt build 的定义是"创建一个 .output 目录，里面是应用、服务器与依赖，已为生产准备好，并报告构建花了多久"。它的开关只有八个：--cwd、--logLevel、--prerender、--target、--dotenv、--envName、-e/--extends、--profile，并把 process.env.NODE_ENV 设为 production。关于目标有三条规则要记牢：--target 给配置过的服务端构建器指定部署目标（文档举的例子是 node-server、vercel、netlify）；--prerender 会始终把 target 设为 static；--preset 仍然作为 --target 的另一个名字被接受，而两者都不传时使用 NITRO_PRESET 与 SERVER_PRESET 环境变量，按这个顺序。想改性能画像就在 v4.4 及以上用 --profile，它会写 nuxt-build.cpuprofile，更早版本只写 CPU profile、没有那份耗时分解。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建
  - target
  - preset
  - 命令行

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


029 我们用 `build` → `preview` 走了一遍本地验证，重点是"别把 preview 当生产"。这一篇回到 `build` 命令本身：**它到底产出什么、`--target` 这个参数在决定什么。**

## 官方对这条命令的定义

> `build` 命令创建一个 `.output` 目录，其中包含你的应用、服务器与全部依赖，已经为生产环境准备好，并报告构建花了多久。

这句话里有三个可以核对的点：

1. **产物目录是 `.output`。** 012 讲过"一次构建产出三样东西"，053 会把这个目录拆开看。
2. **"应用、服务器与全部依赖"——它是自足的。** Server Engine 页对这件事有一句更硬的表述：
   > Nitro 产出一个**独立于 `node_modules`** 的服务器 dist。
   
   这就是"构建完可以把整个仓库扔掉、只带 `.output` 走"的依据。
3. **它会报告耗时。** 所以耗时的异常波动本身就是信号（第 4 节的 `--profile` 是它的深挖版）。

## 用法与全部选项

```bash [Terminal]
npx nuxt build [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--prerender] [--target=<target>] [--dotenv=<path>...] [--envName=<environment>] [-e, --extends=<layer-name>...] [--profile=<verbose>]
```

| 选项 | 官方说明 |
| --- | --- |
| `--cwd=<directory>` | 指定项目根目录 |
| `--logLevel=<silent\|info\|verbose>` | 构建期日志级别 |
| `--prerender` | 构建 Nuxt **并预渲染静态路由** |
| `--target=<target>` | 为配置好的服务端构建器指定部署目标（例如 `node-server`、`vercel`、`netlify`） |
| `--dotenv=<path>...` | 要加载的 `.env`，可重复，靠后的优先 |
| `--envName=<environment>` | 解析配置覆盖时使用的环境（构建时默认 `production`） |
| `-e, --extends=<layer-name>...` | 从某个图层继承 |
| `--profile=<verbose>` | 剖析性能，退出时写 V8 CPU profile 与 JSON 报告；`--profile=verbose` 打印完整控制台报告 |

外加一条 note：

> 本命令把 `process.env.NODE_ENV` 设为 `production`。

## `--target` 到底在选什么：三条规则

这一节是 037 的全部重点。官方给的三条，一条比一条容易忘。

**规则一：`--target` 是"给服务端构建器"的。**

官方原话里有个容易被略过的定语——"**for the configured server builder**"。也就是说它不是 Nuxt 自己在决定，而是交给配置好的那个服务端构建器（088 会讲 `server.builder` 默认是 `@nuxt/nitro-server`）。

**规则二：`--prerender` 会强行改写 target。**

> `--prerender` **始终**把 target 设为 `static`。

**这条要记住，因为它会覆盖你的意图**：`nuxt build --target=vercel --prerender` 里那个 `--target` 是被 `--prerender` 说"不，这次是 static"的对象。想清楚你要的是"预渲染成静态"还是"部署到某个平台"，二者不能靠这条命令同时表达。

**规则三：`--preset` 是旧名字，且不传时看两个环境变量。**

> `--preset` 仍然作为 `--target` 的另一个名字被接受。**当两者都没传时，使用 `NITRO_PRESET` 与 `SERVER_PRESET` 环境变量，按这个顺序。**

这段解决两个常见困惑：

- **"我看到老文章里写 `--preset`，是不是过期了？"** 官方说**仍然接受**，但正式名字是 `--target`。新项目里写 `--target`。
- **"我什么参数都没传，它凭什么知道我是 vercel？"** 靠那两个环境变量（在很多托管平台的构建环境里会自动存在）。这也意味着**同一条 `nuxt build` 在不同平台上可能产出不同 preset** ——复现构建时要把它们钉住。

顺带一个官方在部署页给的默认值：

> （Node.js server preset）未指定或自动检测不到格式时的**默认输出格式**。

以及部署页给的两种显式写法，**它们和 `--target` 是同一件事的不同入口**：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  nitro: {
    preset: 'node-server',
  },
})
```

```bash [Terminal]
NITRO_PRESET=node-server nuxt build
```

对照一下就很清楚了：`NITRO_PRESET` 走的就是规则三那条环境变量通道；`nitro.preset` 是配置入口；`--target` 是命令行入口。**三条路殊途同归，但优先级与"谁覆盖谁"这件事，官方只在 build 页给了"两者都不传才用环境变量"这一句**（至于配置项与命令行谁赢，官方在这里没有明说，我不替它编）。

## `--profile`：它写出的文件分两个版本

官方在 build 页给的那句：

> `--profile` 会把 V8 CPU profile 写到你的项目里的 `nuxt-build.cpuprofile`。**它报告的构建耗时，连同构建目录里的 `perf-report.json` 与 `perf-trace.json`，来自 Nuxt 自己的构建剖析，需要 Nuxt v4.4 或更新版本。**

对照 034 讲的 dev 版（那份写的是 `nuxt-dev.cpuprofile`），命令级的差别只有一处：**dev 那句额外说明"更早版本上 CPU profile 仍然会写出来"，build 这句没有重复这条**。所以别假设两页说明逐字一致。

`--profile=verbose` 是加一份完整控制台报告，官方对 dev 与 build 给的是同一句描述。

**这一项和 043 的 `nuxt analyze` 不是一回事**：`--profile` 量的是"构建过程花了多久、时间在哪"；`nuxt analyze` 量的是"产物里每一块占了多少"。想给包体减肥看后者。

## 自查

- [ ] `nuxt build` 的产物目录是什么？它为什么可以不依赖 `node_modules` 跑？
- [ ] `--prerender` 对 `--target` 做了什么？
- [ ] `--preset` 与 `--target` 的关系，官方的措辞是什么？
- [ ] 什么都不传时，构建目标由哪两个环境变量决定，顺序是什么？
- [ ] `--profile` 与 `nuxt analyze` 各自量什么？

## 下一篇

038 讲 `nuxt generate` 的真实身份——官方一句话就把它的悬念消掉了：它触发的就是带 `prerender` 参数的 `nuxt build`。

## 本文依据

- `api/commands/build`（build 创建 .output 目录包含应用、服务器与全部依赖并为生产准备好且报告构建耗时；用法行与八个选项 --cwd/--logLevel/--prerender（Build Nuxt and prerender static routes）/--target（Deploy target for the configured server builder，例如 node-server、vercel、netlify）/--dotenv 可重复靠后优先/--envName（构建默认 production、dev 默认 development）/-e extends/--profile；note 本命令设 process.env.NODE_ENV 为 production；note "--prerender will always set the target to static"；"--preset is still accepted as another name for --target. When neither is passed, the NITRO_PRESET and SERVER_PRESET environment variables are used, in that order."；"--profile writes a V8 CPU profile to nuxt-build.cpuprofile… the build timings it reports, along with perf-report.json and perf-trace.json in your build directory, come from Nuxt's own build profiling and need Nuxt v4.4 or later"）
- `getting-started/deployment`（Node.js server preset 三条描述，其一为"未指定或自动检测到格式时的默认输出格式"；显式设置预设的两种写法 nitro.preset: 'node-server' 与 NITRO_PRESET=node-server nuxt build）
- `guide/concepts/server-engine`（Standlone Server 一节：Nitro 产出独立于 node_modules 的服务器 dist，Nuxt 在运行 nuxt build 时把它生成到 .output 目录）
- `api/commands/dev`（--profile 写 nuxt-dev.cpuprofile、note 构建耗时与 perf-report.json/perf-trace.json 需 Nuxt v4.4 或更新、更早版本仍写 CPU profile——用于与 build 版对照）
