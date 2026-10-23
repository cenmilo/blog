---
url: /posts/7e9ea32165720bcd9c9f330603fd2c6d/
title: 738 - 目录边界（1）：app、server、shared 谁能自动导入谁
date: 2026-10-20T09:00:00+08:00
updated: 2026-10-20T09:00:00+08:00
author: cmdragon

summary:
  摘要：`useFetch` 不用 import、`defineEventHandler` 不用 import，而你自己写在 `server/services/` 里的函数偏偏要 import——这不是风格问题，是四张自动导入注册表各有扫描范围。这一站放六个探针文件把边界画出来：先看生成的注册表里登记了谁，再看类型检查把没登记的报成哪一行，最后把 `shared/` 那条最容易踩的规矩逐条对上。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 自动导入
  - shared 目录
  - 类型上下文
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 同一件事，三种写法

在 `app/` 里可以直接写 `useFetch(...)`，在 `server/api/` 里可以直接写 `defineEventHandler(...)`，但在 `server/services/` 里自己写的函数，调用方必须老老实实 `import`。仓库里现存的代码就是这个样子：

```ts [server/api/auth/github/callback.get.ts（节选）]
import { writeAudit } from '../../../services/audit'
import { METER_KIND_LOGIN, recordMeterEvent } from '../../../services/metrics'
import { exchangeCodeForIdentity } from '../../../services/auth/oauth'
```

而反过来，`server/api/` 下**没有任何一个文件**从 `../utils/` 里显式导入过东西——那些是被自动登记的。这两件事同时成立，说明自动导入不是一句"框架帮你 import"，而是**有几张表、每张表有自己的扫描范围**。这一站把表读出来。

## 四张注册表，各登记了什么

`nuxt prepare` 每次都会重写它们（上一站量的 mtime 就是这件事的证据）。本轮在六个位置各放一个探针函数，跑一次 prepare，然后在生成的文件里搜这六个名字：

```text
.nuxt/imports.d.ts                    →  export { probeAppComposable } from '../app/composables/probe-app'
                                        export { probeSharedUtil }    from '../shared/utils/probe-b'
.nuxt/types/imports.d.ts              →  const probeAppComposable / const probeSharedUtil（外加 globalThis 里两份 readonly）
.nuxt/types/shared-imports.d.ts       →  const probeSharedUtil
.nuxt/types/nitro-imports.d.ts        →  const probeServerUtil  ← 只有这一张表里有
                                        const probeSharedUtil
六个名字的归属：
  probeAppComposable   （app/composables）    app 侧两张表有，nitro 表没有
  probeServerUtil      （server/utils）       nitro 表有，app 侧没有
  probeSharedUtil      （shared/utils）       三张表都有 ← shared 的意义就在这
  probeSharedNested    （shared/utils/deeper）哪张表都没有
  probeSharedRoot      （shared/ 根下的文件）  哪张表都没有
  probeService         （server/services）    哪张表都没有
```

最后一行是本篇最有用的一条：**`server/services/` 不在任何扫描范围里**。它不是"约定优于配置"的目录，只是你自己建的一个文件夹；要跨文件用里面的东西，就必须显式导入。仓库里那六处 `import ... from '../../../services/...'` 不是历史包袱，是唯一能编译过的写法。

## 类型检查把边界报成行号

光看注册表还不够——真正决定"能不能编译过"的是类型检查。把探针放在一起调用，跑 `npx nuxt typecheck`：

```text
app/utils/probe-app-caller.ts(5,39): error TS2304: Cannot find name 'probeSharedNested'.
app/utils/probe-app-caller.ts(6,37): error TS2552: Cannot find name 'probeSharedRoot'. Did you mean 'probeSharedUtil'?
server/api/probe-caller.get.ts(5,14): error TS2304: Cannot find name 'probeService'.
server/api/probe-caller.get.ts(5,14): error TS2304: Cannot find name 'probeService'.
退出码 2
```

四条里有三点要读出来：

能被自动导入的那三位（`probeSharedUtil`、`probeServerUtil`、`probeAppComposable`）**一个错都没有**，所以这些报错不是"探针写错了"，是边界本身。

同一个"没登记"，TypeScript 给的编号不一样：`TS2304 Cannot find name` 是直说找不到；`TS2552` 那一条是它猜你想用 `probeSharedUtil`。**别把编号当成问题的分类**，看后半句。

`probeService` 那行被打了两遍——四个 tsconfig 各自把服务端项目扫了一遍（app／server／shared／node 是四份独立上下文，上一站讲过）。**同一处代码报两遍不等于有两个问题**，数错误条数之前先按文件行号去重。

## `shared/` 的规矩：只有两个子目录

官方讲 `shared/` 那一节把扫描范围写得很死，只有 `shared/utils/` 和 `shared/types/` 会被自动导入；这两个目录里的**嵌套子目录不算**，除非把目录加进 `imports.dirs` 与 `nitro.imports.dirs`。它给的那棵目录树值得原样抄一遍，四行刚好覆盖四种情况：

```bash [Directory structure]
-| shared/
---| capitalize.ts        # Not auto-imported
---| formatters
-----| lower.ts           # Not auto-imported
---| utils/
-----| lower.ts           # Auto-imported
-----| formatters
-------| upper.ts         # Not auto-imported
---| types/
-----| bar.ts             # Auto-imported
```

也就是说 `shared/utils/x.ts` 里的导出可以在 app 和 server 两边直接调用，而 `shared/utils/formatters/x.ts` 不行——**多一层目录就少一半便利**，这不是玄学，是扫描器只看一层。剩下的文件用 `#shared` 别名引，官方原话是这个别名由 Nuxt 自动配好、"不管从哪里引都一样"：

```ts
import capitalize from '#shared/capitalize'
import lower from '#shared/formatters/lower'
import upper from '#shared/utils/formatters/upper'
```

本轮在 app 侧写了 `import { probeSharedUtil as viaAlias } from '#shared/utils/probe-b'`，类型检查没有对它报任何错（错误清单里没有 `#shared` 这一项）。别名是从生成的 tsconfig 里来的，四份上下文各有多少条 paths 也能量：

```text
app    paths 40 条  含 #shared 2 条、#server 2 条、~/ 有、#imports 有
server paths 35 条  同上
shared paths 39 条  同上
node   paths 19 条  #shared 0 条、#server 0 条、~/ 没有、#imports 没有
其中 #shared/* 指向 ../shared/*
```

node 那份管的是 `nuxt.config.ts`、仓库根的各种 `*.config.ts` 这类文件——**它们不在同一个别名宇宙里**。在配置文件里写 `#shared/...` 或 `~/...` 是解析不到的，要引就用相对路径或 `node:url` 拼。

## 为什么要有这条墙：两个包不是一份

`shared/` 里不能引 Vue 或 Nitro 的代码——官方那节用 important 写着。原因也写在那一节里：Nuxt 打两个独立的包，Vue 应用（客户端加服务端渲染）与 Nitro 服务（接口、服务端中间件、服务端插件），它们分别打包、跑在不同上下文。`shared/` 的代码会同时出现在两边，所以它哪一边都不能引：组件和组合式函数需要 Vue 应用运行时和 Nuxt 上下文（`useNuxtApp()`、`useRoute()`），Nitro 里没有；反过来 Node API、Nitro 工具、服务端路由处理器不能进浏览器，硬引会把服务端逻辑漏进客户端包里。

有一类看着能用但最好别用：**只引类型**。`import type` 在编译期被擦掉，不会把运行时代码拖到另一个包里，所以跨边界引类型"看起来没问题"。官方的建议仍然明确——共用的类型（比如接口的响应形状）放 `shared/types/`，那里两侧都自动导入；只用在一侧的类型放那一侧：`app/types/` 只在 Vue 应用上下文自动导入，`server/types/` 只在 Nitro 上下文自动导入。这么分不是为了整齐，是为了让类型上下文和运行时上下文一一对上（上一站量的那四份 tsconfig 就是这个结构）。

另外记一条时间线：`shared/` 这个目录是 **Nuxt v3.14 起**才有的，从更老版本迁过来的项目里，等价物往往是散在两边各写一份的同名文件。

## 这一步的自查清单

1. 新建目录时先问一句"这张表扫不扫它"：app 侧看 `.nuxt/imports.d.ts`，服务端侧看 `.nuxt/types/nitro-imports.d.ts`，共用看 `.nuxt/types/shared-imports.d.ts`。三个文件都在 `nuxt prepare` 之后才有。
2. `shared/` 里只放 `utils/` 与 `types/` 这两棵子树，且**只放一层**；更深的路径改用 `#shared/...` 显式导入。
3. `server/services/`、`server/repositories/` 这类自建的子目录，调用方一律显式 `import`——它们不在扫描范围，写自动导入只会等到类型检查或运行时才炸。
4. 类型检查报 `TS2304`／`TS2552` 时先怀疑目录位置，而不是名字拼写；两处重复的报错按行号去重。
5. 共用类型进 `shared/types/`，单侧类型留在自己那一侧（`app/types/`、`server/types/`）。
6. 配置文件（`nuxt.config.ts` 与仓库根那几个 `*.config.ts`）用另一套上下文：里面没有 `#shared`、`~`、`#imports`。

| 现象 | 先查什么 |
| --- | --- |
| 明明写了文件，却 `Cannot find name` | 那个目录在不在扫描范围（本篇四张表），或者中间隔了子目录 |
| 同一句在服务端文件里报两遍 | 正常，四份 tsconfig 各自扫过；按文件行号去重 |
| `shared/` 里的文件引了 `useRoute()` 后打包出问题 | 那条 important：shared 不能引 Vue 或 Nitro 的代码 |
| 配置文件里 `#shared/...` 解析不到 | node 上下文里没有这组别名，改相对路径 |
| 想让某个目录也被自动导入 | 官方给的是 `imports.dirs` 与 `nitro.imports.dirs`——两侧要分别配，只配一侧另一边仍然认不出 |

下一步 [739《目录边界（2）：形状只有一份：shared/types 两端同时消费》](/posts/a67ad5f52bb89f6f205bf7cbde439073/)：把类型放进六个不同位置、跑三趟类型检查，看每一侧免 import 到底认不认；`#shared` 什么时候是必需的、node 那一份配置里为什么没有这组别名，也在同一趟里量出来。

## 本文依据

- `shared/` 的用途、只有 `shared/utils/` 与 `shared/types/` 被自动导入、嵌套子目录需 `imports.dirs` 与 `nitro.imports.dirs`、目录树那份注释、`#shared` 别名与三段示例导入、v3.14+ 才有这个目录：`https://nuxt.com/docs/4.x/directory-structure/shared`
- 不能混引 Vue 与 Nitro 代码（两个独立包、不同上下文）、组件与组合式函数需要 Nuxt 上下文、服务端代码进浏览器的三种后果、只引类型的做法与"共用类型放 `shared/types/`"、`app/types/` 与 `server/types/` 各自只在一侧自动导入：`https://nuxt.com/docs/4.x/directory-structure/shared`
- 自动导入的整体机制与文件扫描规则：`https://nuxt.com/docs/4.x/guide/concepts/auto-imports`、`https://nuxt.com/docs/4.x/directory-structure/app/composables#how-files-are-scanned`
- 四份类型上下文（app／node／server／shared）与 `typescript.tsConfig` 的扩展位：`https://nuxt.com/docs/4.x/guide/concepts/typescript`
- 本篇所有读数来自同一个仓库本轮的运行：六个探针文件与两个调用方文件已删除，`nuxt prepare` 重跑后 `.nuxt/imports.d.ts`、`.nuxt/types/imports.d.ts` 里 `probe` 命中 0，`npx nuxt typecheck` 回到 rc=0（输出 0 字节），`npm run lint` rc=0（仍只挂着那条 `vue/no-v-html` 的 warning）
