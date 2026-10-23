---
url: /posts/a14882a69187ab533b599c5829f32885/
title: 763 - 登录链接全链（2）：邮箱契约放在 shared，两侧读的是同一份
date: 2026-10-24T10:00:00+08:00
updated: 2026-10-24T10:00:00+08:00
author: cmdragon

summary:
  摘要：邮箱格式校验写两遍，迟早一份松一份紧——用户看到的就是"填的时候没报错、点提交才失败"。这一站看这份契约怎么只留一处，以及顺着它查出来的一个真问题：换个大小写写法，就能绕过按邮箱计的限流。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 数据校验
  - 限流
  - 邮件登录
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 表单里没报错，点提交才失败

用户在页面上粘了一个前后带空格的邮箱，浏览器里的校验过了，按提交，接口回一句"格式不对"。这种"两套规则不一致"不是能靠小心避免的，它是**写了两个地方的必然结果**。

仓库里这件事只有一份定义：`shared/schemas/auth.ts`（65 行）。它不是工具函数，是一段契约，所以它的落点选择本身就带着规则。

## 契约就四行，剩下的都是边界

```ts [shared/schemas/auth.ts]
export const EMAIL_MAX = 254
export const emailSchema = z.email().max(EMAIL_MAX)
export const loginLinkSchema = z.object({ email: emailSchema })

export function parseLoginLink (input: unknown): EmailResult { /* 归一 → 校验 → 分类 */ }
```

`parseLoginLink()` 是唯一入口，返回一个判别联合：成功给 `{ ok: true, email }`，失败给 `{ ok: false, email: '', issue }`，`issue` 只有 `missing`、`invalid`、`too_long` 三种。文件头写着三条被单测钉住的边界（zod 4.6.7 上的行为，不是推测）：

| 边界 | 行为 | 钉住它的用例 |
| --- | --- | --- |
| 单字符顶级域名 | 被拒（域名段要求至少两个字母），所以 `a@example.c` 过不了 | `auth-schema.test.ts` 第一条 |
| 整串长度 | zod 不管 RFC 的 254 上限，得自己 `.max()`；254 通过、255 报 `too_big` | 同名那条 |
| trim 与大小写 | zod 两件都不做，所以归一必须在进 schema 之前 | 「自己不 trim」那条 |

`test/unit/auth-schema.test.ts` 一共 14 条，本轮跑出来 `14 passed`。第三行是本站的主角，往下看。

## 归一只做两件事，而且键与值不同规则

```ts [shared/utils/normalize-email.ts]
export function normalizeEmail (input: string): string {
  const trimmed = input.trim()
  const at = trimmed.lastIndexOf('@')
  if (at <= 0) return trimmed
  return `${trimmed.slice(0, at)}@${trimmed.slice(at + 1).toLowerCase()}`
}
```

去首尾空白、**只有 `@` 后面那一段**降小写。本地部分原样保留是刻意的（邮箱本地部分按 RFC 是大小写敏感的），单测里有一条就叫「域名降小写，本地部分保持原样」，钉的是 `CmdRagon@Example.COM` → `CmdRagon@example.com`。

产物上的读数是这一条：

```text
[归一] {"探针键":"email:cmdragon@example.com","校验探针":{"accepted":true,"normalized":"CmdRagon@example.com"}}
```

同一个输入，**键里是全小写、值里保留本地部分的大小写**。这两条规则不一样是故意的，理由在下一节。

## 换一种大小写，就能多要几份额度

上一站说过限流是两只桶：邮箱维与 IP 维。邮箱维的键以前是这么拼的：

```ts
plan.push(`email:${email}`)   // 修之前：直接用归一后的值
```

归一保留了本地部分的大小写，于是 `CmdRagon@example.com` 与 `cmdragon@example.com` 落进**两个不同的桶**。而这两个写法在库里是同一个人——`users.email` 与 `login_tokens.email` 都是 citext，比较与唯一约束忽略大小写。**限流按"两种写法"算，账号按"一个人"算**，那封邮件想收多少封就能收多少封：换大小写即可让邮箱维形同虚设。

IP 维还挡着，所以在产物上看不出问题（两维同一份额度，同一台机器连打先撞 IP）。这条只能在单位层测——把 IP 那一维摘掉，只留邮箱维，交替两种写法打四次（桶容量给 3）：

```text
修之前  AssertionError: expected [ true, true, true, true ] to deeply equal [ true, true, true, false ]
        Tests  1 failed | 8 passed (9)
修之后  ✓ test/unit/rate-limit.test.ts (9 tests)   →  Tests  9 passed
```

修法是把键的拼法收进一个函数，并且让它和 citext 用同一条比较规则：

```ts
/** 键只用于计数；落库与发信用的还是归一后的原样地址。 */
export function emailBucketKey (email: string): string {
  return `email:${email.toLowerCase()}`
}
```

`assertRateLimit()` 与探针都用它。顺带修掉的是另一件更隐蔽的事：探针 `rate-check.get.ts` 以前**自己又拼了一遍** `email:${email}`。同一件事两个写入口，就会出现"探针说明明还剩 3 次、接口却直接 429"——它读的根本不是那个桶。现在键只有一个出处，产物上的对照就成立了：

```text
[桶键] {"混合大小写的键":"email:cmdragon@example.com","全小写的键":"email:cmdragon@example.com","剩余额度":[9,8]}
```

两个写法同一个键、额度从 9 掉到 8，说明扣的是同一个桶。这条 `test/e2e/rate-check.spec.ts`（三条，本轮 `3 passed (26.0s)`）是这一层第一次有端到端覆盖——探针接口以前没人测，直到它和限流各写一遍键。

## 从 issue 到人话，中间隔着一张表

`parseLoginLink()` 失败时不给消息、只给分类（`issue`），因为分类才是能被两侧共同消费的那一位。接口拿它拼 `code` 和文案：

```ts
message: EMAIL_COPY[result.issue ?? 'invalid'],
code: `EMAIL_${(result.issue ?? 'invalid').toUpperCase()}`,
```

产物上两种失败的完整读数（`[两类失败]` 摘出来的关键几位）：

| 输入 | 状态码 | `data.code` | `message` |
| --- | --- | --- | --- |
| 没带 `email` 参数 | 400 | `EMAIL_MISSING` | 请填写邮箱地址。 |
| `email=not-an-email` | 400 | `EMAIL_INVALID` | 邮箱地址格式不对，检查有没有 @ 和域名结尾。 |

两句话不一样，但它们来自同一份 `EMAIL_COPY`：**改文案不会改分类，客户端的分支也不会跟着断**——这正是上一线"按 `code` 分支、不去匹配人话"那条规矩在这条链上的落点。还有一个容易写错的地方：`missing` 与 `invalid` 能分开，全靠"参数根本不是字符串时原样往下传"，把没带参捏成空串，`missing` 那条分支就再也走不到（`email-check.get.ts:14-15` 那两行注释写的就是这件事）。

## 四处调用契约，第五处只用归一

| 用它的地方 | 怎么引 | 为什么 |
| --- | --- | --- |
| `app/components/WiringEmailContract.vue` | `import { parseLoginLink } from '#shared/schemas/auth'` | 客户端要用就得显式引 |
| `server/api/auth/login-link.post.ts` | 同上，`#shared` 别名 | 接口的格式闸 |
| `server/api/email-check.get.ts` | 同上 | 契约探针 |
| `server/api/token-demo.post.ts` | 同上 | 演示接口 |
| `server/api/rate-check.get.ts` | `normalizeEmail(...)`，**不写 import** | 它在 `shared/utils/` 下，属于自动导入 |

最后一行与上面四行的差别就是这份契约能只有一份的原因：`shared/utils/` 与 `shared/types/` 会被自动导入，**其它位置（包括 `shared/schemas/`）必须用 `#shared` 别名显式引**。刻意放在 `schemas/` 是为了"谁在用看得见"——`email-check.get.ts` 的头注释把这两行的差别写在了代码里。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 前端放行、接口 400 | 两边各写一套校验 | 契约只放 `shared/schemas/`，两侧都调同一个函数 |
| 带空格的邮箱被判"格式不对" | zod 不 trim | 先 `normalizeEmail()` 再进 schema，顺序别反 |
| 同一个人换个大小写就能多要额度 | 桶键用了保留大小写的值 | 键单独走 `emailBucketKey()`，与 citext 同一条比较规则 |
| 探针说还剩 3 次、接口直接 429 | 键的拼法有两个写入口 | 键只允许一个出处，其余地方调用它 |
| 「没填邮箱」和「填错了」提示一样 | 把缺参捏成了空串 | 非字符串就原样传下去，`missing` 与 `invalid` 分得开 |

## 你自己跑一遍

1. `npx vitest run test/unit/auth-schema.test.ts` → `14 passed`；把 `EMAIL_MAX` 改成 10 再跑，会红两条（254 那条和超长那条），改回来。
2. `E2E_PORT=3140 npm run test:e2e -- test/e2e/rate-check.spec.ts` → `3 passed`，看 `[桶键]` 与 `[归一]` 两行。
3. 想复现被绕过的那次元限流：把 `assertRateLimit()` 里 `emailBucketKey(email)` 改回 `` `email:${email}` ``，跑 `npx vitest run test/unit/rate-limit.test.ts` → `1 failed`，四个都放行；改回来。
4. `curl -s 'http://localhost:<产物端口>/api/email-check?email=%20%20CmdRagon@Example.COM%20%20'`，看回来的 `normalized` 里本地部分还是大写——**值没被削平，削平的是键**。

## 这一站的边界

产物侧的三条读数来自 Playwright 自己起的构建产物（端口 3140，26.0 秒），单位层两条红绿对照来自同一份文件的两次运行；没有连库、没有 SMTP，`.env` 的值没进任何输出。**citext 的"比较忽略大小写"这一条我引的是列定义与建表迁移里的注释，库里的实际行为本轮没有读数**——连库与写库的授权问到第十二次仍无答复，所以那句话在正文里只作为设计依据出现，不作为断言。

三件没做：真发信时收件服务器对本地部分大小写的处理各家不同，本站不动值就是为了让服务商去判；`shared/schemas/auth.ts` 的 `EmailResult` 里失败时 `email` 给空串而不是 `undefined`，这个选择没写进单测（下一站测限流时顺手补一条）；`rate-check` 探针只有三条端到端，桶跨进程不共享这件事它量不到。

仓库这一轮的永久改动：`server/utils/http/rate-limit.ts` 新增 `emailBucketKey()` 并改用它拼邮箱维的键；`server/api/rate-check.get.ts` 改用同一个函数（不再自己拼）并改正了文件头那句与实现相反的说法；`test/unit/rate-limit.test.ts` 从 8 条到 9 条；新增 `test/e2e/rate-check.spec.ts`（3 条）；四个文件的注释清掉指向别处的编号、内部路径与作者口吻说法。

## 本文依据

- 契约本体与三条边界：本仓库 `shared/schemas/auth.ts`、归一函数 `shared/utils/normalize-email.ts`
- 限流的两只桶与键的收口：`server/utils/http/rate-limit.ts`、`server/api/rate-check.get.ts`
- 单位层 14 条与 9 条：`test/unit/auth-schema.test.ts`、`test/unit/rate-limit.test.ts`；端到端 3 条：`test/e2e/rate-check.spec.ts`
- `shared/utils/` 与 `shared/types/` 才自动导入、其它位置要 `#shared` 显式引：<https://nuxt.com/docs/4.x/guide/directory-structure/shared>
- citext 列与"比较忽略大小写"的列定义：`db/tables/custom-types.ts`、`db/tables/users.ts:14`、`db/tables/login-tokens.ts:10`（表侧的规则见 [752《用户表与会话表（3）：login_tokens 按邮箱记账，为什么它没有外键》](/posts/728d8115969a9ff035b153d953b8a4a2/)）
- 上一篇（五个落脚点与十步顺序）：[762《登录链接全链（1）：一条链五个落脚点，先看清谁调谁》](/posts/d37c40ed992de15b14e135f2307b7e63/)
- 下一篇：[764《登录链接全链（3）：入口那三步的顺序：格式、令牌桶、才发信》](/posts/9f3ca2df0afb2d865e506acb1dd6bb1c/)
- 本轮读数：`[桶键]`、`[归一]`、`[两类失败]` 三行原文，修之前 `1 failed | 8 passed` 与之后 `9 passed`，`3 passed (26.0s)`、`14 passed`
