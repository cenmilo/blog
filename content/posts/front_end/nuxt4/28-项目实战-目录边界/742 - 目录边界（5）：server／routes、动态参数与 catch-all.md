---
url: /posts/4da4e73a736020732f24a8defec04d7b/
title: 742 - 目录边界（5）：server／routes、动态参数与 catch-all
date: 2026-10-20T13:00:00+08:00
updated: 2026-10-20T13:00:00+08:00
author: cmdragon

summary:
  摘要：`server/routes` 里的文件不带 `/api` 前缀，这一点谁都记得。真正容易出事的是另外两处：方括号参数在服务端目录里能不能省、catch-all 会不会把你没写的地址都接走。这一站同一份命名在两边各放一个文件，用真请求打出十四档读数——可选参数在服务端目录里不是"可省"，而 catch-all 的范围比文档写的那样窄。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 服务端路由
  - 动态参数
  - catch-all
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先记一个数

`.nuxt/types/nitro-routes.d.ts` 里那张表（上一站量过），站里现在真实注册的是 **22 条路径键、22 个方法位**。这一站往里加六个探针文件，数字变成 **28 条**，读完再全删掉，回到 22。为什么要先记这个数：下面所有结论都是对着这张表读的，而表里多一条少一条，只看屏幕是看不出来的。

这一站要问三件事，都不在"前缀"那条显然的规则里：

- `[[id]]` 这种"可以省掉这一段"的写法，在 `server/` 目录里还成立吗？
- `[...slug].ts` 与 `[...].ts` 这两种 catch-all，各注册成什么地址、真的会接走一切未命中吗？
- 服务端的路由参数，`getRouterParam` 与 `event.context.params` 两种取法给的是一样的东西吗？

## 三句话的官方口径

`server/api` 里的文件自动带 `/api` 前缀；不想带前缀就放 `server/routes`。这是同一条节里的第一句，紧接着还有一句容易跳过的注：**服务端路由对动态路由的支持不如 pages 完整**——它没说什么地方不完整，这一站就是要把这句量出形状。

参数写法是方括号文件名（`server/api/hello/[name].ts`），取值两种：`getRouterParam(event, 'name')`，或者 `event.context.params`；文档还建议用 `getValidatedRouterParams` 配一个 schema 把参数校验一并做掉。

catch-all 两种写法：`server/api/foo/[...].ts` 里段落读 `event.context.params._`；具名形式 `server/api/foo/[...slug].ts` 读 `event.context.params.slug`。文档对第一种的说法是"注册一条接走所有未匹配路由"——这句字面上挺吓人，本轮专门验了它的范围。

## 六个探针注册成了什么

加完文件跑一次 `npx nuxt prepare`，逐行读生成物（左边地址、右边方法位）：

```text
/probe-plain              default     ← server/routes/probe-plain.ts
/probe-nested/deep        get         ← server/routes/probe-nested/deep.get.ts
/probe-opt/:[id]          default     ← server/routes/probe-opt/[[id]].ts
/api/probe-user/:id       get         ← server/api/probe-user/[id].get.ts
/api/probe-ca/**:slug     default     ← server/api/probe-ca/[...slug].ts
/api/probe-anon/**        default     ← server/api/probe-anon/[...].ts
```

三行 `server/routes` 的没有 `/api`，三行 `server/api` 的都有——目录决定前缀，文件名决定后面那截。方法位分布也从 22 条时的一组变成 `delete` 1、`get` 14、`post` 4、`default` 9。

`/probe-opt/:[id]` 那一行是这一站的第一个异常：`:[id]` 不是可选参数该有的形状。vue-router 里"可以省"写作 `:id?`，而这里参数名带着一对方括号。对照就更好玩了——同一份命名放页面那边：

```text
app/pages/probe-opt/[[id]].vue   →  /probe-opt/:id?   名字 probe-opt-id
server/routes/probe-opt/[[id]].ts →  /probe-opt/:[id]  方法位 default
```

第一行是从路由记录里读出来的（`router.getRoutes()`，上一站那招），第二行是 Nitro 的注册表。同一个 `[[id]]`，页面那边变成 `:id?`，服务端这边变成字面量 `:[id]`。

## 真请求十四档

e2e 那一层自己把 `npm run build` 的产物拉起来打（1 passed，17.0s）：

```text
GET /probe-plain          → 200 | {"from":"server/routes","suffix":"none"}
GET /api/probe-plain      → 404
GET /probe-nested/deep    → 200 | {"from":"server/routes","suffix":"get"}
PUT /probe-nested/deep    → 404
GET /api/probe-user/123   → 200 | {"viaRouterParam":"123","viaContext":"123"}
GET /probe-opt            → 200 | 一段 HTML（页面，不是接口的 JSON）
GET /probe-opt/xyz        → 200 | {"shape":"optional","params":{"[id]":"xyz"},"path":"/probe-opt/xyz"}
GET /api/probe-ca/a/b/c   → 200 | {"shape":"named-catch-all","slug":"a/b/c"}
GET /api/probe-ca         → 404
GET /api/probe-anon/x/y   → 200 | {"shape":"anon-catch-all","underscore":"x/y"}
GET /api/totally-unknown  → 404
GET /totally-unknown      → 404
GET /api/count            → 200 | 10
GET /api/docs             → 200 | 一段清单，第一条 slug 是 pricing-faq
```

逐条读，六个结论。

**前缀是双向的**：`/probe-plain` 通、`/api/probe-plain` 不通。想给一个地址去掉 `/api`，只能挪目录，不能在原地改名。

**后缀在无 `api` 目录里同样生效**：`PUT /probe-nested/deep` 是 404。上一站那条"未匹配的方法掉进页面渲染兜底"在这里一模一样复现，跟有没有 `/api` 前缀无关。

**两种取法给同一个值**：`{"viaRouterParam":"123","viaContext":"123"}`。区别只在类型——`event.context.params` 是个可空的记录，`getRouterParam` 直接给你那一位。

**`[[id]]` 在服务端不是"可省"**：`/probe-opt/xyz` 命中的是接口，而它读到的参数键是字面量 `"[id]"`（`params: {"[id]":"xyz"}`）。这意味着 `getRouterParam(event, 'id')` 永远取不到值——名字里那对方括号得带上，而没人会这么写。至于 `/probe-opt` 为什么回了 200：那是页面那边那条 `:id?` 接住了，返回的是一段 HTML 文档，不是接口的 JSON。同一个地址两位竞争，接口那一位没匹配上。这就是"不如 pages 完整"的具体形状。

**catch-all 的范围比文档字面窄**：`/api/probe-anon/x/y` 命中了那条 `[...]`，而 `/api/totally-unknown` 仍然是 404。注册表里那行写的是 `/api/probe-anon/**`——它只接自己目录下的路径，不是全站兜底。另外 `/api/probe-ca`（一个段落都不给）是 404：`**` 要至少一段才算命中。

**参数值是字符串而不是数组**：`/api/probe-ca/a/b/c` 读到的 `slug` 是 `"a/b/c"`。pages 那边的文档例子给的是数组（`["hello","world"]`），两边口径不同，别照搬。

最后两行是回归位：`/api/count` 与 `/api/docs` 都还是 200，说明这几个探针没把站里现成的接口盖住。

## 一处写法差别，顺手记下

同一个 `[slug]`，两套表里写法不同：

```text
Nitro 的注册表      /api/docs/:slug
pages 的路由记录    /docs/:slug()
```

上一站读页面时看到的是带括号的 `:slug()`，这一站读接口表时是不带括号的 `:slug`。你手打这两个字符串去比对时别当成 bug。

## 目录一深，就轮到 `#server`

参数目录套两层以上，相对路径就要数 `../../..`，这是文档专门给别名理由的场景。那条节（v4.3 起）给的例子正好是一个带参数的深目录：

```ts [server/api/users/[id]/profile.ts]
// 不要这样数层数：
// import { formatUser } from '../../../utils/formatUser'

// 用 #server 别名，它指回 server/ 目录：
import { formatUser } from '#server/utils/formatUser'
```

站里已经在用这一位——`server/api/console/metrics.get.ts` 引 services 那行就是 `import { metricsForUser } from '#server/services/metrics'`，而这个文件本身在 `server/api/console/` 下，按相对路径得数两层。两句配套的限制也要记住：别名**只能在 `server/` 里面用**，客户端里 import `#server` 会报错；而且它属于 Nuxt 给的那套上下文，单测的 node 那一层解析不到它（上一站那条 `Cannot find module` 就是同一件事），所以被单测直接引的服务端文件照旧走相对路径。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 接口死活 404，页面能打开 | 地址前缀搞错：`server/routes` 不带 `/api` | 对着 `nitro-routes.d.ts` 搜一遍自己的文件名 |
| `getRouterParam(event, 'id')` 一直是空 | 服务端目录里的 `[[id]]` 把键注册成了 `[id]` | 别在服务端目录用双括号；文档没给这一位的替代写法 |
| catch-all 读到的段落是 `"a/b/c"` 而不是数组 | 服务端与页面两边口径不同 | 自己按 `/` 切，或改用 `event.context.params` 直接看形状 |
| 加了 `[...]` 之后担心别的接口被吞 | 它只接自己目录下、且至少一段 | 打一次未命中的地址确认（本轮那条 `/api/totally-unknown` 仍是 404） |
| 未匹配的 PUT 请求返回一张 HTML 错误页 | 请求掉进页面渲染的兜底 | 非 `/api` 的地址本来就是这个行为，别按接口的错误体解析 |
| 想在服务端做参数校验 | 文档建议 `getValidatedRouterParams` 配 schema | 站里 `shared/schemas/` 已经放着一份 zod，直接接 |

## 自己跑一遍

1. 建 `server/api/probe-one/[id].get.ts`，把 `[id]` 打回响应体，`npx nuxt prepare` 后在 `.nuxt/types/nitro-routes.d.ts` 里搜 `probe-one`，你会看到 `'/api/probe-one/:id'`。
2. 打 `GET /api/probe-one/abc`，确认两种取法都给 `abc`。
3. 在 `server/routes` 下建 `probe-two/[[id]].ts`，重跑 prepare：地址变成 `/probe-two/:[id]`——方括号进参数名了。打 `GET /probe-two/x` 看响应里的 `params`，你会亲眼见到那个带方括号的键。
4. 建 `server/api/probe-three/[...slug].ts`，打 `/api/probe-three/a/b/c`（读到字符串）与 `/api/probe-three`（404），再打一个你根本没写的 `/api/nothing-here`（仍是 404，说明它没接管全站）。
5. 全部删掉，重跑 `npx nuxt prepare`，数一下注册表：条数要回到你开工前那个数（这个仓库本轮是 22 条路径键、22 个方法位）。

八个探针文件（六个服务端、一个页面、一条 e2e spec）与一个测试文件都已删净，空目录回收；复跑 `nuxt prepare` 后注册表回到 22／22、里面 `probe` 命中 0，`npx nuxt typecheck` **rc=0、输出 0 字节**，`npm run lint` **rc=0**（那条 `vue/no-v-html` 的 warning 还在），Nuxt 那一层 5 个测试文件 14 条用例全过。

下一步 [743《目录边界（6）：server／services 不自动注册：业务代码为什么要显式 import》](/posts/5133b7136338d1b1e583a9f24e8b7958/)：同目录树里 `server/utils` 能裸用、`server/services` 非写 import 不可；再往下还有一层——类型检查放行的裸写法，在单测里直接 `ReferenceError`。这条线到那一站收尾。

## 本文依据

- `server/api` 自动前缀 `/api`、`server/routes` 不带前缀、"服务端路由的动态支持不如 pages 完整"那条注：`https://nuxt.com/docs/4.x/directory-structure/server`
- 路由参数的 `[name].ts` 写法、`getRouterParam` 与 `event.context.params` 两种取法、`getValidatedRouterParams` 加 schema 的建议：`https://nuxt.com/docs/4.x/directory-structure/server`
- catch-all 两种写法（`[...].ts` 读 `params._`、`[...slug].ts` 读 `params.slug`）与它接住未匹配路径的说法：`https://nuxt.com/docs/4.x/directory-structure/server`
- 方法后缀与"其它方法回 405"那句（本轮在产物上读到的是 404，上一站已有读数）：`https://nuxt.com/docs/4.x/directory-structure/server`
- `#server` 别名（v4.3 起）指回 `server/` 目录、深目录里不必数 `../../`、只能在 `server/` 内用而客户端引用会报错：`https://nuxt.com/docs/4.x/directory-structure/server`
- pages 侧的可选参数 `[[slug]]` 与动态参数写法：`https://nuxt.com/docs/4.x/directory-structure/app/pages`
