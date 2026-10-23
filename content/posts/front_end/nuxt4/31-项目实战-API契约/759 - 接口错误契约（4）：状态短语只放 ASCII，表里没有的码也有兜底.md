---
url: /posts/cbe7ef8c83f5341303324e49e39f01ea/
title: 759 - 接口错误契约（4）：状态短语只放 ASCII，表里没有的码也有兜底
date: 2026-10-23T12:00:00+08:00
updated: 2026-10-23T12:00:00+08:00
author: cmdragon

summary:
  摘要：状态行那句短语不是自由文本，官方把它限制在可见 ASCII 之内，而框架遇到越界字符的做法是删掉、不是报错。这一站从一张状态码表里查出一处缺行：同一个 503，两条出口发出两种短语，客户端读状态行就会看见分叉。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 错误处理
  - HTTP 状态行
  - 类型检查
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 同一个 503，两条状态行不一样

同一台机器、同一个构建产物，两个接口都回 503：

```text
HTTP/1.1 503 Error
HTTP/1.1 503 Service Unavailable
```

前一条是发登录链接那个入口（没配 SMTP 时的降级），后一条是点链接验票那个入口（没配库时的降级）。状态码相同，响应体里那四个字段的形状也相同，**只有状态行那句短语分叉了**。

分叉的原因很小：`STATUS_TEXT` 那张表里**没有 503 这一行**。本轮补上了，补之前先量到了红。

## 先说清"状态行"是哪一段

一条 HTTP 响应的开头是一行三段：协议版本、三位状态码、一句 reason phrase。

```text
HTTP/1.1 503 Service Unavailable
└─┬─┘    └┬┘ └────────┬────────┘
 协议     码        那句短语
```

这一站在 Nuxt 里叫 `statusMessage`，在浏览器与 `curl` 眼里就是那句短语。它和响应体里的 `message` 是**两个东西**：前者受字符集约束、给人一眼扫过去；后者是 UTF-8 的 JSON 字符串，写中文完全合法，前面几站那句"服务器暂时无法处理这个请求，请稍后重试"就在 `message` 里。

官方对这句短语的限定写得很直白：

> The `statusText` property is intended for short, HTTP-compliant status texts (e.g., "Not Found"). It should only contain horizontal tabs, spaces, and visible ASCII characters (`[\t\u0020-\u007E]`).

## 这张表就是这一位的唯一来源

走 `throw` 的错误，短语一律来自 `server/utils/http/api-error.ts` 里这张表；本轮补了最后一行，现在是九行：

| 状态码 | 短语 |
| --- | --- |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 409 | Conflict |
| 422 | Unprocessable Entity |
| 429 | Too Many Requests |
| 500 | Internal Server Error |
| 503 | Service Unavailable |

表里没有的码不会崩，兜成 `'Error'`（`STATUS_TEXT[statusCode] ?? 'Error'`），单测里用 418 钉着这一位：

```text
✓ test/unit/api-error.test.ts (10 tests) 10ms
```

`message` 与 `code` 那两位随你写，`statusMessage` 这一位不给你自由发挥的空间——它的字符集是定的。

## 分叉是怎么量出来的

本站的契约用例里新加了一条，让**同一个 503 从两条出口各走一遍**：`POST /api/auth/login-link`（没配 SMTP，走 `throw apiError`）与 `GET /api/auth/verify?token=<64 位十六进制>`（没配库，走 `apiFailureResponse`）。这一趟环境里既没有 SMTP 也没有连接串，两个入口正好停在"配置缺失"那一位。

补表之前：

```text
[503 对照] {"throw":{"状态":503,"状态行":"Error","statusMessage":"Error","code":"MAIL_NOT_CONFIGURED"},"自回":{"状态":503,"状态行":"Service Unavailable","statusMessage":"Service Unavailable","code":"DB_NOT_CONFIGURED"}}
  1 failed
  8 passed (26.1s)
```

补上 `503: 'Service Unavailable'` 之后：

```text
[503 对照] {"throw":{"状态":503,"状态行":"Service Unavailable","statusMessage":"Service Unavailable","code":"MAIL_NOT_CONFIGURED"},"自回":{"状态":503,"状态行":"Service Unavailable","statusMessage":"Service Unavailable","code":"DB_NOT_CONFIGURED"}}
  9 passed (24.8s)
```

为什么值得修：`data.code` 两边一直是对的（`MAIL_NOT_CONFIGURED` 与 `DB_NOT_CONFIGURED`，本来就该不同），**分叉只出现在状态行**。而状态行是 `curl`、浏览器控制台、各种网关日志、以及"按状态码记指标"的东西唯一能看到的位。给同一件事两种写法，早晚有人拿它当区分依据。

顺带把这条纪律落到单测里（新增那条 503 用例，附了原因注释）：**表要跟着实际用到的状态码补齐**，缺行不报错，只是安静地兜成 `Error`。

## 框架不拦你，它把越界的字符删掉

那句限定不是靠自觉执行的。h3 1.15.11 里做这件事的只有三行：

```js
const DISALLOWED_STATUS_CHARS = /[^\u0009\u0020-\u007E]/g;   // index.mjs:577
function sanitizeStatusMessage(statusMessage = "") {          // :578
  return statusMessage.replace(DISALLOWED_STATUS_CHARS, "");  // :579
}
```

`setResponseStatus()` 在把值挂到响应之前调它（同一文件 `:762`）。注意这条正则与官方那句里的字符类是同一个东西——而它的动作是**替换成空**，不是抛错。所以中文短语不会让请求失败，只会让状态行变成空串：这是一个静默降级，比报错难发现。

三条单测把这条钉住（跑出来的就是上面那个 `10 tests`）：

```ts
expect(sanitizeStatusMessage('服务不可用')).toBe('')            // 整句中文 → 空
expect(sanitizeStatusMessage('Service 不可用 503')).toBe('Service  503')  // 混合串 → 只留 ASCII，空格原位
```

第三条是个循环：把表里九个码各造一次错误，断言每个短语过一遍过滤之后**一个字都不掉**。这条用例是防将来有人往表里加一句中文——那种改动不会让任何请求报错，只会让某条状态行突然变空。

## 顺手把这个错误对象的类型口径读准

`apiError()` 里那个 `createError` 不写 `import`（服务端自动导入）。它的返回类型是 `NuxtError`，不是 h3 的 `H3Error`。这一位以前只是"标注会红"，本轮把红趟的原文读全了。临时探针两行：

```ts
import type { H3Error } from 'h3'
export const probeA: H3Error = createError({ statusCode: 400, message: '甲' })
export const probeB: string = createError({ statusCode: 400, message: '乙' })
```

`npm run typecheck` 的读数（`rc=2`，三行报错）：

```text
server/utils/http/probe-errtype.ts(5,14): error TS2322: Type 'NuxtError<unknown>' is not assignable to type 'H3Error<unknown>'.
  Types of property 'statusCode' are incompatible.
    Type 'number | undefined' is not assignable to type 'number'.
server/utils/http/probe-errtype.ts(8,14): error TS2322: Type 'NuxtError<unknown>' is not assignable to type 'string'.
server/utils/http/probe-errtype.ts(8,14): error TS2322: Type 'H3Error<unknown>' is not assignable to type 'string'.
```

第一行才是重点：**不兼容的具体原因是 `statusCode` 的可空性**（`NuxtError` 上是 `number | undefined`，h3 那一侧要求 `number`），不是"两个同名类型长得不一样"这种含混说法。后两行同位置各报一次，说明那个位置上有两个 `createError` 的声明参与推断（Nuxt 的那一个与 h3 的那一个），所以写 `import { createError } from 'h3'` 与不写，拿到的东西并不一样——上一站讲的"服务端自动导入登记指向哪一个"在这里再次现形。

探针删掉之后 typecheck 回到 `rc=0`。这个结论也回写进了 `api-error.ts` 的文件头注释（原来那句只说"会被退回 TS2322"，现在写的是报错原文与可空性这条真原因）。

## 站里现在这一位写得多规矩

用同一把尺子量了两批字面值：

| 数的是 | 条数 | 含非 ASCII 的 |
| --- | --- | --- |
| `statusMessage` 的字面值 | 17 | 0 |
| `message` 的字面值 | 28 | 28（全部） |

第二行是**正对照**：如果扫描不工作，两行都会是 0。同一条正则在一批数据上命中 28 次、在另一批上命中 0 次，才说明"短语那一批真的干净"。

17 条字面值对应 18 个调用点，差的那一个是因为 `verify.get.ts` 把 401 那一组抽成了共享常量 `LINK_INVALID`，两处引用同一份定义——这正是它该被抽成一个常量的理由：两条出口共用的形状只写一遍。

还有一位数：走 `throw` 的调用点用过的状态码是 400×9、401×1、403×2、429×1、500×1、503×2，**每一个都在表里**（503 是本轮补的）。表里目前没人用的两位是 404 与 409——留着不亏，它们对应的码在别的接口上是标准动作。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| `curl -i` 看到 `HTTP/1.1 503 Error`，而另一条路由是 `Service Unavailable` | 那张表缺行，兜底值接管了 | 把用到的码补进表；新加状态码时同一条 e2e 对照用例扩一行 |
| 状态行整句变成空（`HTTP/1.1 503 `） | 短语里写了非 ASCII 字符，被过滤删空 | 中文只写 `message`，短语用英文短串 |
| 按状态短语做分支的代码在某个接口上失灵 | 同码两个出口发出不同短语 | 客户端按 `data.code` 分，不读状态行 |
| 类型检查报 `not assignable to type 'H3Error'` | 返回值是 `NuxtError`，它的 `statusCode` 可空 | 别按 `H3Error` 标注；要显式类型就写 `NuxtError` 或让推断走 |
| 明明写了中文短语，测试全绿 | 过滤是删字符不报错，没有断言撞它 | 加一条"每个短语过过滤后原样不变"的循环用例（本文第三节那条） |

## 你自己跑一遍

1. `npx vitest run test/unit/api-error.test.ts` → `10 passed`。然后把 `api-error.ts` 里 503 那一行删掉重跑，本轮真跑过一次这个组合：

   ```text
   AssertionError: expected 'Error' to be 'Service Unavailable'
   Tests  1 failed | 9 passed (10)
   ```

   注意**只有那一条专用用例红**——最后那条九码循环不红，因为兜底值 `'Error'` 本身是合法 ASCII。改回来。
2. `E2E_PORT=3122 npm run test:e2e -- test/e2e/api-contract.spec.ts` → `9 passed`，看 `[503 对照]` 那行里两个 `状态行` 是否相同。
3. 亲眼看一次状态行：`curl -s -o /dev/null -D - -X POST http://localhost:3122/api/auth/login-link -H 'content-type: application/json' -d '{"email":"nobody-9f2c@example.com"}' | head -1`，再对 `curl -s -o /dev/null -D - 'http://localhost:3122/api/auth/verify?token=<64 个 a>' | head -1`。
4. 试一次静默降级：把表里某一行改成中文短语，跑第 1 与第 2 步——第 1 步会红那条循环用例，第 2 步的状态行会变成空串。
5. `grep -rhoE "statusMessage: '[^']*'" --include=*.ts server/ | wc -l` 数你自己的短语字面值，再用同一条管道加 `| grep -cP '[^\x00-\x7F]'` 看有没有越界的。

## 这一站的边界

两条 503 的对照读数来自 `nuxt build` 的产物服务（端口 3122，绿趟 24.8 秒、红趟 26.1 秒），环境里既没有 SMTP 也没有数据库连接串，`.env` 的任何值都没出现在输出里；`verify` 那一条停在"形状对但没配库"的 503，没有走到任何一句 SQL。**连库/写库的授权问到第八轮仍无答复**，所以凡是"库里真有一条券"的读数一律留给读者。

两处没做：HTTP/2 之下没有 reason phrase 这件事本文没量（本站产物跑在 node-server 的 HTTP/1.1 上）；`import { createError } from 'h3'` 与不写之间还有哪些可观察差别，本轮只读到 `statusCode` 可空性这一条。

仓库这一轮的永久改动：`STATUS_TEXT` 补一行 `503`；`test/unit/api-error.test.ts` 从 6 条到 10 条（503 那条＋字符集那三条）；`test/e2e/api-contract.spec.ts` 从 8 条到 9 条（两条出口的 503 对照）；`api-error.ts` 文件头两处注释按本轮读数改写（字符集那条补上"删字符不是报错"，类型那条补上报错原文，并删掉一处读者看不到的内部路径引用）。临时探针 `server/utils/http/probe-errtype.ts` 用完已删，`npm run typecheck` 回到 `rc=0`。

## 本文依据

- 那张表、兜底值与两处注释：本仓库 `server/utils/http/api-error.ts`
- `statusText` 的字符集限定：<https://nuxt.com/docs/4.x/getting-started/error-handling>
- 过滤的实现（`DISALLOWED_STATUS_CHARS`、`sanitizeStatusMessage`、`setResponseStatus`）：`node_modules/h3/dist/index.mjs:577-580`、`:754-764`，h3 版本 1.15.11
- 十条单测：`test/unit/api-error.test.ts`；九条产物契约：`test/e2e/api-contract.spec.ts`
- 两条 503 出口各自的场景：`server/services/auth/login-link.ts`（`MAIL_NOT_CONFIGURED`）、`server/api/auth/verify.get.ts`（`DB_NOT_CONFIGURED`）
- 上一站（两条出口与响应头不同价）：[758《接口错误契约（3）：throw 还是自己回响应，差别在查询串里有没有凭据》](/posts/bfa776071905ef16a29a234b510ddb65/)
- 下一篇：[760《接口错误契约（5）：客户端按 code 分支，不去匹配人话》](/posts/46d497d1113ae82a298fb98776367a10/)
- 本轮读数：补表前后两条 `[503 对照]` 原文、`1 failed／8 passed` 与 `9 passed (24.8s)`、`10 passed` 与 `6 passed` 的单测差、三行 TS2322 报错原文、17 条短语与 28 条 message 的正负对照
