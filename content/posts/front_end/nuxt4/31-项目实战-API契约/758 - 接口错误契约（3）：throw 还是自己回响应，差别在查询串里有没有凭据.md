---
url: /posts/bfa776071905ef16a29a234b510ddb65/
title: 758 - 接口错误契约（3）：throw 还是自己回响应，差别在查询串里有没有凭据
date: 2026-10-23T11:00:00+08:00
updated: 2026-10-23T11:00:00+08:00
author: cmdragon

summary:
  摘要：报错有两种发法：把错误抛给框架，或者自己把响应写完。六个键的形状一模一样，差别在一位——框架那条路会把完整请求地址原样放进响应体，未处理的异常还把它写进日志。地址里带着登录凭据的路由因此不能走那条路。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 错误处理
  - 登录凭据
  - 响应头
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 一切都对，只有多出来的那一个键不对

你写了"点链接登录"那条路由，地址长这样：`/api/auth/verify?token=一串一次性凭据`。用户点了个过期链接，你按上一站的规矩发一条 401：

```ts
throw apiError(event, { statusCode: 401, message: '这条登录链接无效，或者已经被用过了。', code: 'LINK_INVALID' })
```

状态码对、`message` 是人话、`data.code` 齐、号也在。但响应体里还有一个键是框架替加的：

```json
"url": "http://localhost:3117/api/error-case?kind=bad-request&token=canary-token-value"
```

**框架把完整请求地址装进了 `url`**，连那枚刚刚被你拒收的凭据一起。这份响应把它又发回去了一次——转发的旧邮件、分享出去的活动记录、留了历史的公司代理，都算。

同一层还有第二个出口，专门给这种路由用。两个出口怎么选，这一站讲清楚，并且把差别量出来。

## 两条出口各是什么

```ts
// 一：交给框架的错误通道
throw apiError(event, { statusCode, message, code })

// 二：自己把响应写完
return apiFailureResponse(event, { statusCode, statusMessage, message, code })
```

发法不同：第一条 throw 出去，由框架去序列化；第二条是个普通返回值，`apiFailureResponse()` 自己 `setResponseStatus`、自己 `setHeader`、自己拼那个对象。第二条的入参多一位 `statusMessage`，因为它是自己写头，不再走那张状态短语表。

**六个键一模一样，`data` 里那两位也一模一样，只差一位：`url` 是谁填的。**

```ts
// server/utils/http/api-error.ts 里那一行的全部秘密
url: getRequestURL(event).pathname,   // 只到路径为止，查询串不进来
```

## 框架那条路的代码就三十行

`throw` 那一路最终走到 Nitro 的错误处理器（发布产物用的是 `nitropack/dist/runtime/internal/error/prod.mjs`，下面带行号引它）。和这一站有关的是四段：

```js
const isSensitive = error.unhandled || error.fatal;                      // :18
const url = getRequestURL(event, { xForwardedHost: true, xForwardedProto: true });  // :21

if (isSensitive && !opts?.silent) {                                      // :34
  console.error(`[request error] ${tags} [${event.method}] ${url}\n`, error);  // :36
}

const body = {                                                           // :54
  error: true,
  url: url.href,                                    // :56 ← 完整地址，连查询串
  statusCode, statusMessage,
  message: isSensitive ? "Server Error" : error.message,   // :59
  data: isSensitive ? void 0 : error.data,                 // :60
}
```

三件事一次说清：

- `url` 这一位**永远**是完整地址（`:56`），跟错误是不是你造的都无关。
- `:59-60` 这两行就是上一站那两条读数的出处：走封装的错误是"已处理"的，`message` 与 `data` 原样保住；裸抛的算未处理（`unhandled`），`message` 换成兜底串 `"Server Error"`、`data` 直接给 `void 0`——所以上一站那张表里"裸抛的 500 连 `data` 键都没有"不是猜的，是这行写的。
- `:36` 那一行日志只有未处理或致命错误才会打，**但它带的是同一个完整地址**。凭据在查询串里时，这条路由的每一次意外异常都会在日志里留一枚凭据。

## 同一个地址、两种出口的对照读数

本站的探针接口加了一条分支，让两种出口从**同一条路由、同一个查询串**过去，剩下的一切都相同（同一套中间件、同一条 `routeRules`）。假凭据用的是一个看得见词义的字符串 `canary-token-value`。

```text
[两出口对照] {"throw状态":400,"throw的url":"http://localhost:3117/api/error-case?kind=bad-request&token=canary-token-value","自回状态":401,"自回的url":"/api/error-case","自回的键":["error","url","statusCode","statusMessage","message","data"],"自回体里含凭据":false}
```

左边那一串里 `token=canary-token-value` 出来了；右边 `url` 只剩 `/api/error-case`，`自回体里含凭据` 是 `false`——**整份响应体里搜不到这 18 个字符**。右边那个 `data.requestId` 仍然等于头里的号，也就是说自回并没有丢掉上一站那条契约。

再补一条最坏的：凭据在查询串里、错误又是裸抛的。

```text
[裸抛带凭据] {"status":500,"url":"http://localhost:3117/api/error-case?kind=raw&token=canary-token-value","message":"Server Error","有data键":false}
[WebServer] [request error] [unhandled] [GET] http://localhost:3117/api/error-case?kind=raw&token=canary-token-value
```

第二行是产物服务自己打到 stderr 的。**响应体和日志各留一份**，这就是"带凭据的路由不许走异常通道"的全部理由——不是怕用户看到 500，是怕这两处各留一枚凭据。

## 响应头不同价：走框架那条是白拿三个

`throw` 那一路由框架写头，`apiFailureResponse()` 只写了两位（`content-type` 与 `x-content-type-options`）。同一趟测试里把两条出口的响应头都读出来：

| 响应头 | 走 `throw` | 自己回 | 是谁给的 |
| --- | --- | --- | --- |
| `content-type: application/json` | 有 | 有 | 前者是框架，后者是自己 `setHeader` |
| `x-content-type-options: nosniff` | 有 | 有 | 同上（框架那行在 `prod.mjs:42`） |
| `x-frame-options: DENY` | 有 | 没有 | 框架（`:44`） |
| `content-security-policy: script-src 'none'; …` | 有 | 没有 | 框架（`:48`） |
| `referrer-policy: no-referrer` | 有 | 没有 | 框架（`:46`） |
| `cache-control: no-store` | 有 | 有 | 谁都不是这两条出口给的：是 `routeRules` 里以 `/api/` 打头那条通配规则 |
| `x-request-id` | 有 | 有 | 发号中间件 |

`cache-control` 那一行值得停一下：两边都有，但功劳不在出口。框架那条路是"已经有人写过就不写"（`:51-52`，它补的是 `no-cache`），本站的路由规则先一步把 `no-store` 铺了上去；自回那条路自己不写，也被同一条规则兜住。**把那一行路由规则删掉，两条出口的缓存行为就分叉了**——这才是"自己回要自己把关头"的具体含义。

真实路由是怎么补的：`server/api/auth/verify.get.ts` 与两条 GitHub 路由在函数开头就自己写了两行——

```ts
setHeader(event, 'cache-control', 'no-store')
setHeader(event, 'referrer-policy', 'no-referrer')
```

`x-frame-options` 与那条 CSP 没补，也不该补：那两个头是给会被浏览器当文档渲染的 HTML 错误页准备的，这里的响应是 `application/json` 加 `nosniff`，浏览器不会执行它。**要知道自己少了什么，比机械照抄更重要。**

## 什么时候必须自己回

本站 `apiFailureResponse()` 用了 18 处，分布在 6 个文件，逐个数过来就知道判据长什么样：

| 文件 | 处数 | 地址里带着什么 | 为什么不走 `throw` |
| --- | --- | --- | --- |
| `server/api/auth/verify.get.ts` | 5 | `?token=` 一次性登录凭据 | 失败是日常（重放、过期、别人转发的旧链接），且这两处都会留下凭据 |
| `server/api/auth/github/callback.get.ts` | 7 | `?code=` 与 `?state=` | 同上，而且一次带两个 |
| `server/api/auth/github/start.get.ts` | 1 | `?redirect=` | 与回调共用一套判定，不另立形状 |
| `server/utils/runtime-secret.ts` | 1 | 被上面两条引用 | 配置项类型不对要 503 加一行日志，不该冒成内部错误 |
| `server/api/docs.get.ts` | 1 | 无凭据 | 内容目录读不到是部署问题，明确回 503，别把 `ENOENT` 那行打进日志 |
| `server/api/docs/[slug].get.ts` | 3 | 无凭据 | 同上一条 |

判据三条，命中任意一条就走自回：**查询串里有凭据**；**失败是这条路由的日常**；**你要自己定形状**（比如 503 之外还要往日志里写一行 `level: 'error'`）。三条都不沾、只是"请求写错了"的一次性 4xx，`throw` 更省事——少写两个 `setHeader`，还白拿三个安全头。

## 摘掉那一位，看用例红不红

`api-error.ts` 里那一行从 `.pathname` 改成 `.href`（也就是让自回那条路也发完整地址）：

```text
1 failed  7 passed (26.3s)
  > 144 |  expect(selfBody.url).toBe(CASE_PATH)
```

打出来的那一行也跟着变了：`"自回的url":"http://localhost:3119/api/error-case?kind=self-send&token=canary-token-value"`。改回原样是一趟全绿：`8 passed (25.0s)`。红的只有这一条，另外七条（号同值、栈特征零命中、裸抛形状、头不同价）都不依赖这一位，所以照旧绿——**一条断言该在哪里红，本身就是它在钉什么的证据**。

顺带说一句上一站那条读数在源码里的位置：开发服务用的是另一个文件（`error/dev.mjs`），里面是 `message: error.message`（`:79`，不脱敏）和 `stack: error.stack?.split(...)`（`:81`，上一站那个多出来的键）。这两个函数是 Nitro 的错误处理器，只有异常冒到框架才会被调用，所以不 throw 的那条路根本不进它们——"自回的六个键在开发服务和产物里一样"就是这条代码路径上的推论，本轮没有再跑开发服务，想证它只要打同一条地址数一遍键。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 日志里出现带 token 的整条地址 | 那条路由把失败 `throw` 出去了 | 查询串带凭据的路由改用 `apiFailureResponse()` |
| 自己回的响应少三个安全头 | 它不经过框架错误通道（本轮读了头对照那两行） | 要就在路由开头自己 `setHeader`，不要就写明理由 |
| 失败响应被缓存住了 | 以为出口会给 `cache-control`，其实是路由规则给的 | 规则之外的路径要在路由里自己写 `no-store` |
| 换了发法之后 `statusMessage` 变成空 | 自回那条要自己给 `statusMessage`，不查那张状态短语表 | 调用时补上，或者让它也走表（`STATUS_TEXT` 是同一份） |
| 客户端拿不到号 | 自回时没读 `event.context.requestId` | 函数内部已经搬了号，别在调用处自己造 |
| dev 上看不到问题、线上凭据漏了 | dev 的 `message` 不脱敏，形状也和产物不同 | 泄露判断只做在 `nuxt build` 的产物上 |

## 你自己跑一遍

1. `E2E_PORT=3117 npm run test:e2e -- test/e2e/api-contract.spec.ts`，看 `8 passed` 与两条自打印读数（`[两出口对照]`、`[头对照]`）。端口占用就换一位。
2. 把整趟输出重定向到文件，再搜 `canary-token-value`：会命中三处——测试打印的两条 url，加上产物服务那一行 `[request error]`。第三处就是日志侧的泄露面。
3. 摘行验证：`server/utils/http/api-error.ts` 第 103 行 `.pathname` 改成 `.href`，重跑第 1 步，确认红的是那一条；改回来。
4. 开发服务上数一遍键：`npm run dev -- --port 3120`，另开终端 `curl -i -s 'http://localhost:3120/api/error-case?kind=self-send&token=abc'`，数响应体的顶层键（应当还是六个，没有 `stack`）。
5. `grep -rn "apiFailureResponse(" server/` 数一遍你自己仓库里的处数与文件。

## 这一站的边界

所有读数来自 `nuxt build` 的产物服务（绿趟端口 3117、红趟 3119），探针接口不碰数据库，本轮也没有连库；`/api/auth/verify` 那条真实路由这次没直接打——在没有连接串的环境里它必先撞上"没配库"的 503，那个形状在会话那一篇已经读过。`.env` 里的任何值都没出现在上面任何一段输出里。

两处**没有读数**：`getRequestURL(event, { xForwardedHost: true, xForwardedProto: true })` 里那两个选项在真实反向代理后面会把 `url` 换成外部域名还是保留内网地址（部署环境上自己看一眼，别照抄本文）；开发服务上自回那条路的键数（本轮没跑，上面已标为推论）。

仓库这一轮的永久改动：`server/api/error-case.get.ts` 多一条 `kind=self-send` 分支（因此它的兜底提示文案多列一种，上一站引的那条读数已同步改过）；`test/e2e/api-contract.spec.ts` 从五条用例扩到八条；`server/api/auth/verify.get.ts`、`github/callback.get.ts`、`server/utils/runtime-secret.ts`、`server/api/docs.get.ts` 四个文件的注释里，那些指向别处的编号引用换成了函数名与包内行号。`api-error.ts` 的实现一行没动（摘坏已还原，第 103 行仍是 `.pathname`）。

## 本文依据

- 两条出口与 `url` 那一位：本仓库 `server/utils/http/api-error.ts`
- 框架错误通道的行为（`url.href`、`isSensitive` 两支、五个白拿的头）：`node_modules/nitropack/dist/runtime/internal/error/prod.mjs:18-60`；开发模式那一支：同目录 `dev.mjs:79-81`
- 八条契约与两组对照读数：`test/e2e/api-contract.spec.ts`
- 六个调用点的场景与注释：`server/api/auth/verify.get.ts`、`server/api/auth/github/callback.get.ts`、`server/api/auth/github/start.get.ts`、`server/utils/runtime-secret.ts`、`server/api/docs.get.ts`、`server/api/docs/[slug].get.ts`
- 错误对象与状态码的官方说明：<https://nuxt.com/docs/4.x/getting-started/error-handling> 与 <https://nuxt.com/docs/4.x/api/utils/create-error>
- 上一站（号同值、dev 与产物的形状差异）：[757《接口错误契约（2）：一个请求一个号，头里和体里必须是同一个》](/posts/7933edb88a85f1a1b4d584f9353b1259/)
- 下一篇：[759《接口错误契约（4）：状态短语只放 ASCII，表里没有的码也有兜底》](/posts/cbe7ef8c83f5341303324e49e39f01ea/)
- 本轮读数：两条出口的 url 与键集合、探针值在三处的出现、七个响应头的前后对照、`8 passed` 与摘行后的 `1 failed`
