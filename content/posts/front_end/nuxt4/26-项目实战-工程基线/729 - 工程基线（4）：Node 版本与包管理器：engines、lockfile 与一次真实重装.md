---
url: /posts/a84cfdb58daaff6ae84f184fd57a8e46/
title: 729 - 工程基线（4）：Node 版本与包管理器：engines、lockfile 与一次真实重装
date: 2026-10-18T12:00:00+08:00
updated: 2026-10-18T12:00:00+08:00
author: cmdragon

summary:
  摘要：这一站把"版本"这件事拆开看它到底拦不拦人。`engines` 默认只发一行警告，加上 `--engine-strict` 才给出退出码 1；`packageManager` 这一位 npm 根本不读，读过它的是 corepack；lockfile 记的是上次算出来的那棵树，所以 `npm ci` 面对两种"不一致"的反应完全不同。每一条都在一个四个文件的最小工程上跑过，命令与回执照抄。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - engines
  - lockfile
  - 包管理器
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 你钉住的其实只有依赖名

上一篇构建完，产物 1.96 MB。那条命令真正吃进去的还有两样你没钉住的东西：跑它的 Node 是哪一个，以及这次装出来的依赖树长什么样。把它们交给别人（换一台机器、换一个 CI 镜像）的时候，症状通常不是"装不上"，而是装得上、跑得起来、构建报一条你本地从没见过的错。

这一站把三个钉版本的位逐个量一遍，看它们各自到底拦不拦人。下面所有命令都在上一篇那个最小工程里跑的（`package.json`、`nuxt.config.ts`、`app/app.vue`、`node_modules`），环境是 Node v24.11.1 / npm 11.17.0。

## 官方那句话比包自己声明的宽

讲安装的那一节写的是一句人话：**Node.js 22.x 或更新**，并建议用当前的 active LTS；后面的补充说明里还有一条——尽量用偶数版本（22、24 这种），因为奇数版本是不长期支持的开发线。

装上之后，`package-lock.json` 里 `nuxt` 自己声明的范围是另一回事：

```text
node_modules/nuxt        engines.node = ^22.19.0 || ^24.11.0 || >=26.0.0
node_modules/vite        engines.node = ^20.19.0 || >=22.12.0
node_modules/nitropack   engines.node = ^20.19.0 || >=22.12.0
node_modules/@nuxt/kit   engines.node = >=18.12.0
node_modules/h3          没有声明
node_modules/vue         没有声明
```

两个数放一起就看出口径的差：**22.18 满足"22.x"，但不满足 `^22.19.0`。** 而且这不是一个要求，是一堆——这份 lockfile 的 662 个条目里，389 项声明了自己的 `node` engines。你的项目真正需要的版本是这 389 个范围的交集，而没人会去手算交集。

这台机器上只有一个 Node，没有换成 20.x 再跑一遍，所以下面只讲能量出来的那部分：**这些声明默认不会拦住你。**

## engines 默认只是一行警告

先把配置读出来，再故意把根项目的 `engines` 改成一个不可能满足的范围：

```bash
npm config get engine-strict
# false
```

```json [package.json]
{
  "engines": { "node": ">=99.0.0" }
}
```

然后正常安装。回执里只多了一段警告，**退出码 0，依赖照样装完**：

```text
npm warn EBADENGINE Unsupported engine {
npm warn EBADENGINE   package: undefined,
npm warn EBADENGINE   required: { node: '>=99.0.0' },
npm warn EBADENGINE   current: { node: 'v24.11.1', npm: '11.17.0' }
npm warn EBADENGINE }
```

同一份 `package.json`，加一个开关再装：

```text
npm error code EBADENGINE
npm error engine Unsupported engine
npm error engine Not compatible with your version of node/npm: undefined
npm error notsup Not compatible with your version of node/npm: undefined
npm error notsup Required: {"node":">=99.0.0"}
npm error notsup Actual:   {"node":"v24.11.1","npm":"11.17.0"}
退出码 = 1
```

`package:` 那一格是 `undefined`，因为这次被检查的对象是项目根自己。结论很直白：`engines` 是写在仓库里的声明，`engine-strict` 是每台机器或每个镜像上的配置。后者不在仓库里，所以你提交代码时钉不住它——同事的机器默认值还是 `false`。要让版本判断真的生效，只有两条路：要么在镜像和开发文档里带上 `npm config set engine-strict true`，要么把这条判断写进项目自己的检查脚本（这一篇最后一节就做这件事）。

## packageManager 这一位，npm 压根不读

`package.json` 里还有一个 `packageManager`，形如 `"npm@11.17.0"`。把它改成一个不存在的版本试试：

```json [package.json]
{
  "packageManager": "npm@99.0.0"
}
```

```text
npm ls --depth=0        → 正常列出 nuxt / vue / vue-router，退出码 0，没有任何提示
npm install --dry-run   → 退出码 0，同样一句提示都没有
corepack npm --version  → Error: Server answered with HTTP 404 when performing the
                          request to https://registry.npmjs.org/npm/-/npm-99.0.0.tgz
```

同一条声明，两个程序的反应相反：**npm 不读它，corepack 读它**（本机 corepack 0.34.2）。讲社区贡献的那一节确实解释了这件事：项目 `package.json` 里的 `packageManager` 是给 Corepack 用的，它会替你装上那个版本并用它执行命令，例如 `"pnpm@7.5.0"`；讲开发容器的那一节直接在 Dockerfile 里写 `RUN npm i -g corepack && corepack enable`。

所以这一位的真实作用是：只有在装它的机器已经 `corepack enable` 之后才生效。没启用，它就是一段给同行看的说明文字。项目要么在镜像和上手文档里明确"先跑一次 `corepack enable`"，要么就别指望这一行替你换包管理器。

顺带一条容易上当的：`npm install --dry-run` 听起来是"只看不动"，但它照样跑了 `postinstall: nuxt prepare`——`.nuxt/tsconfig.json` 的修改时间（12:57:09）晚于那次改 `package.json` 的时间（12:56:58）。**dry-run 不保证没有副作用**，验证想法时别把它当成沙箱。

## lockfile 记的是树，不是意图

```text
package-lock.json → lockfileVersion: 3 / packages 条目 662 个 / 9,164 行 / 320 KB
根条目里同时存着：dependencies = {"nuxt":"^4.5.2","vue":"^3.5.44","vue-router":"^5.4.0"}
                  engines      = {"node":">=22.0.0"}
```

`package.json` 写的是能接受的范围，lockfile 记的是**上次实际算出来的那棵树**——每个包的具体版本、它自己的依赖、下载地址和完整性校验。两者谁服从谁，决定了 `npm install` 和 `npm ci` 的行为差别。

## 一次真实重装：npm ci 把自己删干净再装

想知道"按 lockfile 复现"是什么意思，先往 `node_modules` 里放一个自己的文件当标记：

```bash
touch node_modules/_ci-marker.txt
time npm ci
test -f node_modules/_ci-marker.txt && echo 还在 || echo 已被删掉
```

```text
npm ci 退出码 = 0        耗时 10 秒（本地缓存已热）
标记文件：已被删掉
重装后 node_modules 顶层条目 427 个（其中 28 个是 @scope 目录）
唯一包路径（npm ls --all --parseable 去重）587 个
node_modules 总大小 194 MB
```

**`npm ci` 自己会删掉整个 `node_modules` 再按 lockfile 重建**，不需要你手工删除。退出码是 0，十秒这个数建立在缓存已经有全部包的前提下；缓存空的机器上同一棵树要从 registry 重新下载，落盘之后就是那 194 MB。

这里有一条会让人误判的读数。这个最小工程里：

```text
npm ls --all              → 1,374 行 / 587 个唯一包路径
npm ls --all --omit=dev   → 1,374 行 / 587 个唯一包路径（逐行相同）
```

**"去掉开发依赖"砍掉了 0 个包。** 原因不神秘：这份根 `package.json` 里一行 `devDependencies` 都没有，nuxt、vue、vue-router 全在 `dependencies` 里，它们的依赖自然全被算成生产依赖。等你真的加了测试框架、lint、类型工具，这一位才开始起作用——在那之前，"用 `--omit=dev` 缩小镜像"这个动作在这个形状的工程项目里什么也没缩小。

还有一条会吓到人的输出：`npm ls --all` 里有 **150 行 `UNMET OPTIONAL DEPENDENCY`**，名字包括 `cac`、`commander`、`@parcel/watcher`、`@vitejs/devtools`、`@azure/app-configuration`。这条命令的退出码仍然是 0。可选依赖没装是正常状态——那是给可选集成准备的（文件监听的原生扩展、各家配置中心、部署平台的 SDK），看到这一行不代表树坏了。

## 两种"不一致"，只有一种会响

给根 `package.json` 各加一项依赖，再跑 `npm ci`：

| 加的那一项 | lockfile 里有没有 | `npm ci` 的结局 |
| --- | --- | --- |
| `"devalue": "^5.0.0"` | 有（本来就在树上） | 退出码 0，照常装完 |
| `"left-pad": "^1.3.0"` | 完全没有 | 退出码 1，直接拒绝 |

第二次的回执值得逐行读：

```text
npm error code EUSAGE
npm error
npm error `npm ci` can only install packages when your package.json and package-lock.json
npm error are in sync. Please update your lock file with `npm install` before continuing.
npm error
npm error Missing: left-pad@1.3.0 from lock file
```

机制就一句话：**`ci` 只按 lockfile 装，它不替你解析。** 新加的那一项如果 lock 里已经算得出位置，就装得上；如果 lock 里压根没有这一项，`ci` 不会偷偷去联网算一棵新树——它拒绝执行，因为这会破坏"复现"这个前提。所以 `package.json` 一改就要把新的 lockfile 一起提交，这是同一个动作的两半。

真觉得树已经乱到要重造，`nuxt` 自己给了一条命令：

```bash
npx nuxt upgrade --force
```

讲这条命令的那一节写得很清楚：它更新 `package.json` 里的版本、用你的包管理器重装、并清掉构建缓存；`--force` 是**从空白重建 lockfile 和 `node_modules`**，`--dedupe` 在包管理器支持时顺带去重。这比"手工删两个目录再 install"多做的事是把版本更新和缓存清理绑在同一步里。

## lockfile 要提交，node_modules 不要

方向相反的两位：`package-lock.json` 必须进版本库，`node_modules/` 必须不进。讲 `node_modules` 的那一节只有一句要紧的话——这个目录要加进 `.gitignore`，免得把依赖推上去。讲 `.gitignore` 的那一节直接给了推荐名单，原话是**至少**要有下面这些：

```bash [.gitignore]
# Nuxt dev/build outputs
.output
.data
.nuxt
.nitro
.cache
dist

# Node dependencies
node_modules

# Logs
logs
*.log

# Misc
.DS_Store

# Local env files
.env
.env.*
!.env.example
```

分成四组看：前六个名字是开发和构建落在本地的产物；`node_modules` 是依赖树；`logs` 与 `*.log` 是日志；最后三条是本地环境文件——`.env` 和 `.env.*` 都不许进，再用 `!.env.example` 单独放行一份样板，让人知道要配哪些变量名。

这份名单值得逐条抄，理由就在前两节的实验里：`node_modules` 和 `.output` 都能随时重造（前一节那个标记文件、上一篇往 `.output` 里塞的 MARKER，一次重装、一次构建就没影了），而 `.env` 里那几行配置重造不出来，也不该让别人拿到。反过来，lockfile 是"重造得出同一棵树"的唯一凭据，它必须提交。

## 把 engines 真的执行一次

正式项目 `Substrate` 的检查入口 `npm run check` 第一项就是版本判断，做法是把声明变成代码：

```js [scripts/check.mjs（节选）]
// package.json 的 engines.node 写的是 ">=22.0.0"，但那只是声明：
// npm 不装 engine-strict 就不会拦人，所以这条声明在这里被真正执行一次。
const MIN_NODE_MAJOR = 22

function runNodeCheck () {
  const version = process.versions.node
  const major = Number(version.split('.')[0])
  if (major < MIN_NODE_MAJOR) {
    console.error(`FAILED: node >= ${MIN_NODE_MAJOR}（当前 v${version}）`)
    return false
  }
  console.log(`=== node >= ${MIN_NODE_MAJOR} ===\nv${version}`)
  return true
}
```

本轮跑出来的六项：

```text
=== node >= 22 ===        ok    node        rc=0       0 ms
                        ok    lint        rc=0    3026 ms
                        ok    typecheck   rc=0    7047 ms
                        ok    test:unit   rc=0    1627 ms
                        ok    build       rc=0   24413 ms
                        ok    test:nuxt   rc=0    3913 ms
                        全部通过
```

这条判断只看主版本号，所以 Node 22.0 会放过去，而 lockfile 里 `nuxt` 自己声明的范围其实要求 22.19 以上。要不要把次版本也比上，是收紧闸门的一次取舍——放到讲 CI 配置那一站再定。

## 这一步的自查清单

1. `node -v` 落在**偶数且不低于 22**的版本上；`npm -v` 也记一下，出问题时要报这两个数。
2. `package.json` 里有 `engines.node`，且它的下限不宽于你在 lockfile 里看到的 `node_modules/nuxt` 那一条。
3. `package-lock.json` **必须提交**。看到 `Missing: … from lock file` 时先跑 `npm install` 再重跑 `npm ci`，别去删 lock。
4. 写了 `packageManager` 就在上手文档里配一句 `corepack enable`，否则那行等于注释。
5. 把版本判断写进项目自己的检查脚本，别依赖每台机器上的 `engine-strict`。
6. 想知道装得对不对，用 `npm ls --all` 看结构、用退出码看结论；`UNMET OPTIONAL DEPENDENCY` 不算红。

| 现象 | 先查什么 |
| --- | --- |
| 别人 `npm install` 成功、构建报错 | 两人的 `node -v`；再看 lockfile 有没有被各自改乱 |
| `npm ci` 报 `EUSAGE` | `package.json` 与 lockfile 不同步，跑一次 `npm install` 补齐 |
| 明明写了 `engines`，低版本照样装 | `npm config get engine-strict` 是不是 `false`（默认就是） |
| 换了包管理器后依赖装不出来 | `packageManager` 需要 corepack 启用才生效；没启用就按你手上那个包管理器走 |
| 装完少了一大堆东西，怀疑装坏 | 看是不是可选依赖没装，或 lockfile 被删过导致重新解析 |

下一步 [730《工程基线（5）：配置分层：默认值、环境覆盖块与 envName》](/posts/1ada736ce467d9006c014a15c88dea40/)：`nuxt.config.ts` 之外还有谁会参与配置，`$production`、`$env.staging` 这几块什么时候被选中——上一站那句"dev 与 build 读到的不是同一份配置"就是它引起的。

## 本文依据

- Node.js 版本要求（22.x 或更新、建议 active LTS、用偶数版本）：`https://nuxt.com/docs/4.x/getting-started/installation`
- `package.json` 的最小形状与 `postinstall: nuxt prepare`：`https://nuxt.com/docs/4.x/directory-structure/package`
- `nuxt upgrade` 的行为与 `--force` / `--dedupe` / `--channel`：`https://nuxt.com/docs/4.x/api/commands/upgrade`
- `packageManager` 字段与 Corepack 的关系、`corepack enable` 只需一次：`https://nuxt.com/docs/4.x/community/contribution`
- 开发容器里启用 corepack、按 lockfile 安装（`--frozen-lockfile`）：`https://nuxt.com/docs/4.x/guide/best-practices/devcontainers`
- `node_modules/` 与 `.output` 等目录要进 `.gitignore`：`https://nuxt.com/docs/4.x/directory-structure/node_modules`、`https://nuxt.com/docs/4.x/directory-structure/gitignore`
- 升级 Nuxt 与 lockfile 的关系（`--force` 重建）、nightly 频道要删 lockfile 重装：`https://nuxt.com/docs/4.x/getting-started/upgrade`、`https://nuxt.com/docs/4.x/guide/going-further/nightly-release-channel`
- 本篇数字来自仓库根下的最小试验工程与正式项目 `Substrate/` 的同一轮运行，两者的 Node 与 npm 版本相同
