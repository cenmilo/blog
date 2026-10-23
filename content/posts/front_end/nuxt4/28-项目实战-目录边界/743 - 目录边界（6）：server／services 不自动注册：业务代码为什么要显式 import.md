---
url: /posts/5133b7136338d1b1e583a9f24e8b7958/
title: 743 - 目录边界（6）：server／services 不自动注册：业务代码为什么要显式 import
date: 2026-10-20T14:00:00+08:00
updated: 2026-10-20T14:00:00+08:00
author: cmdragon

summary:
  摘要：同一个 handler 文件里，`server/utils` 的函数不 import 就能用，`server/services` 的非写 import 不可——差别不是风格，而是自动导入只认目录名。这一站把这条界线量到最后一层：类型检查放行的裸写法，在单元测试那一层直接 `ReferenceError`，因为 node 上下文里根本没有自动导入这回事。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 自动导入
  - server 目录
  - 分层
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 两行挨着的代码，两种待遇

`server/api/docs.get.ts` 里有这两行，位置相邻、待遇不同：

```ts [server/api/docs.get.ts 的真实两行]
import { listDocs } from '../services/content'      // 第 6 行：必须写
// ...
return apiFailureResponse(event, {                  // 第 17 行：没 import 也能用
```

`listDocs` 住在 `server/services/content/`，`apiFailureResponse` 住在 `server/utils/http/api-error.ts`。为什么一个要写 import 一个不用？答案不在文件名、也不在导出方式，只在一件事：**自动导入的名单是按目录名给的**。

## 三个词

**Server Utilities**：文档那一节讲的东西，口径很窄——"`~~/server/utils` 目录里可以自己加 helper"，给的例子是在 `server/utils/handler.ts` 定义一个包一层响应的 `defineWrappedResponseHandler`，然后在 `server/api/hello.get.ts` 里不 import 直接用。整节没提过别的自建目录。

**`server/services/`**：这个仓库自己起的一层，放"业务与 SQL 不进 handler"的落点。它不在任何名单上，所以它的导出对自动导入来说是隐形的。

**单测层**：`test/unit/` 那批用例跑在普通 Node 环境里，没有 Nuxt 运行时。这一层不认识任何自动导入的名字——这是本站最后那对读数的来源。

## 名单里到底有谁

本轮对着生成物数的（探针删干净之后的状态）：

```text
.nuxt/types/nitro-imports.d.ts   提到 server/utils 的行：45
                                 提到 server/services 的行：0
```

再看仓库这边：`server/services/` 下 15 个文件，引用它们的显式 import **28 处**——`#server/services` 那种写法只有 1 处，其余 27 处是相对路径。反过来，`server/utils` 里 10 个文件，全站对它的显式 import 只有 1 处：

```ts [server/services/auth/verify.ts:11 的真实一行]
import { SESSION_MAX_AGE_SEC } from '../../utils/http/session-cookie'
```

这一行看着像"忘了它是自动导入的"。它不是遗漏，原因下一节用一对读数说清。

## 一次红趟把三条界画出来

探针 handler 里同时写四种用法：

```ts [server/api/probe-svc.get.ts]
import { metricsForUser } from '#server/services/metrics'

export default defineEventHandler(async (event) => {
  const user = requireUser(event)                        // server/utils：不 import
  const viaAlias = await metricsForUser(user.id)         // 显式 #server 别名
  const bareServicesCall = await summarizeMeterRows([])  // server/services：不 import
  const failureShape = apiFailureResponse(event, { /* ... */ })  // server/utils：不 import
  return { viaAlias, bareServicesCall, failureShape }
})
```

`npx nuxt typecheck` → `rc=2`，报错清单只有两行，而且是同一个名字：

```text
server/api/probe-svc.get.ts(6,34): error TS2304: Cannot find name 'summarizeMeterRows'.
server/api/probe-svc.get.ts(6,34): error TS2304: Cannot find name 'summarizeMeterRows'.
```

四条用法各自的答案都在这份清单里：`requireUser` 与 `apiFailureResponse` 一次不报（`server/utils` 自动导入到服务端每一处）；`#server/services/...` 那条显式 import 不报（v4.3 那个别名指回 `server/`，深目录不必数 `../../`）；只有裸写的 services 函数报，而且报了两遍——服务端文件会被不止一份类型上下文扫到，这条上一站讲过。

`summarizeMeterRows` 是 `server/services/metrics/index.ts` 的真实导出。把它挪进 `server/utils/` 再试，同一行立刻不报——名单是按目录给的，不是按文件、也不是按里面写了什么。

## 类型放行了，也照样跑不起来

这一对是本站真正要留下来的东西。`server/services/probe-nodeps.ts` 裸用 `server/utils` 的常量：

```ts [server/services/probe-nodeps.ts]
export function probeReadHeader () {
  return CSRF_HEADER
}
```

`npx nuxt typecheck` 的清单里没有它——**类型检查放行**（`CSRF_HEADER` 确实注册进了服务端那套自动导入）。但同一份文件被单元测试直接引用时：

```text
vitest run test/unit/probe-nodeps.test.ts   →  退出码 1
ReferenceError: CSRF_HEADER is not defined
```

把这行改成显式导入，同一趟用例就绿（退出码 0、1 条用例过）。差别只在多写一行 import。

原因就是前面那句：`test/unit/` 那一层是普通 Node，没有 Nuxt 运行时，也没有任何人往全局里塞名字。自动导入只活在 Nuxt 给你生成的那套上下文里。所以**只要一个服务端文件会被单测直接 import，它内部用到的任何东西——包括同目录树里 `server/utils` 的东西——都得写显式 import**。`SESSION_MAX_AGE_SEC` 那一行就是这个规矩的落地，不是风格洁癖。

顺带一条同族的读数：`#shared`、`#server` 这些别名在 node 那一层也解析不到（上一站那条 `Cannot find module '#shared/utils/format'`）。所以这一层的写法只有一种：相对路径。

## 为什么要留一层"必须显式 import"

自动导入省事，省事的地方也容易看不见。`server/services/` 这一层的规矩是刻意的：

- SQL 与业务判断不进 handler，handler 只做"确认是谁在问、然后调一层"。这一条分工在 `server/api/console/metrics.get.ts` 那种文件里一眼能读出来。
- 名字要能被搜到。想知道"改这个函数会牵动哪几处接口"，grep 一句 `#server/services` 或相对路径就能列出 28 处；如果它是自动导入的，全站的调用点都没有一行 import，就搜不出这份清单。
- 单测能直接引。业务函数要能在 Node 那层单独跑（不需要起 Nuxt），前提就是它自己不依赖任何自动导入的名字。

对照 `server/utils`：那里的东西是"给 handler 与中间件随手用的小工具"，跨很多文件反复用、又不指望在 Node 层单测，自动导入的正合适。两种待遇不是谁更规范，是分工不同。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 类型检查报 `Cannot find name` 而函数就在隔壁目录 | 那个目录不在自动导入名单里（自建目录都不在） | 写显式 import，或者用 `#server/...` 别名 |
| 类型检查全绿，单测 `X is not defined` | 单测层是普通 Node，没有任何自动导入 | 给被测文件里用到的每个名字补 import |
| `#server/services/xxx` 在单测里 Cannot find module | 别名属于 Nuxt 上下文，node 那层没有 | 单测走相对路径 |
| 想找一个函数被谁调用，搜不到几处 | 它靠自动导入，调用点没有 import 行 | 这正是自建 services 层的理由：让入口显式 |
| 同一条报错出现两遍 | 服务端文件被多份类型上下文各扫一次 | 按文件加行号去重再数 |

## 这条线走完，六步各留一句

| 编号 | 一句话结论 |
| --- | --- |
| 738 | 自动导入按目录名给：只有 `shared/utils`、`shared/types`、`app/composables`、`app/utils`、`server/utils` 这些在名单上，且只扫直接一层 |
| 739 | 共用形状只有 `shared/types/` 两端都免 import；`app/types` 与 `server/types` 文档那样写、本版不认 |
| 740 | 页面文件名变路由记录；布局文件名变一张类型表里的键，写错名字当场红 |
| 741 | `server/api` 自动带 `/api`；方法后缀认得很少几位，写错它就变成地址的一部分 |
| 742 | `[[id]]` 在服务端不等于可省；catch-all 只接自己目录下、至少一段 |
| 743 | `server/services` 不在名单里，靠显式 import；类型放行不代表单测跑得起来 |

本轮三个探针文件（一个 handler、一个 services 文件、一个单测）用后即删，`.nuxt` 三张注册表里 `probe` 命中 0、路由表回到 22 条；复跑 `npx nuxt typecheck` **rc=0、输出 0 字节**，`npm run lint` **rc=0**（那条 `vue/no-v-html` 的 warning 还在），单测 **25 文件 184 用例全过**，Nuxt 那一层 **5 文件 14 用例全过**。

下一步 [744《开发库与第一份迁移（1）：连接串有两个名字，各归各的主》](/posts/d14544c58864c9a0189dc4d61e68ff4e/)：`DATABASE_URL` 给迁移命令行、`NUXT_DATABASE_URL` 给跑起来的站点，三趟真请求证明它们互相顶替不了——这条线之外，新的一条线也从这里开工。

## 本文依据

- `server/utils` 是加 helper 的位置、`defineWrappedResponseHandler` 的例子、以及在 `server/api/hello.get.ts` 里不 import 直接用：`https://nuxt.com/docs/4.x/directory-structure/server`
- "只有 `server/utils` 会被自动导入到 `server/` 目录"那条 `::important`（与 `app/utils` 只在应用侧的对照）：`https://nuxt.com/docs/4.x/directory-structure/app/utils`
- `#server` 别名（v4.3）指回 `server/` 目录、只能在 `server/` 内使用：`https://nuxt.com/docs/4.x/directory-structure/server`
- 自建目录与"按目录名注册"的对照读数来源（`server/services` 与 `shared/types` 的边界）：`https://nuxt.com/docs/4.x/directory-structure/shared`
- 自动导入的整体机制与扫描规则：`https://nuxt.com/docs/4.x/guide/concepts/auto-imports`
- 三层测试各跑在什么环境（单测是普通 Node）：`https://nuxt.com/docs/4.x/getting-started/testing`
