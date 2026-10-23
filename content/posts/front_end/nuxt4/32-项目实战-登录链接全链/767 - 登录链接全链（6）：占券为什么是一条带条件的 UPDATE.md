---
url: /posts/d129bfe13d7484d4322e650f512262e9/
title: 767 - 登录链接全链（6）：占券为什么是一条带条件的 UPDATE
date: 2026-10-24T14:00:00+08:00
updated: 2026-10-24T14:00:00+08:00
author: cmdragon

summary:
  摘要：点开的第二条链接为什么没反应，答案是占券那条 UPDATE 里的三个条件。但这一站翻出来的是一处真漏：库连不上时错误没人接，那条为了藏住凭据而写的路由，把明文凭据原样写进了服务端日志与响应体。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 邮件登录
  - 错误处理
  - 凭据管理
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先是日志里掉出明文

一台配了库但库连不上的机器上，点开邮件里的链接，浏览器拿到 500，而服务端那侧多出一行：

```
[WebServer] [request error] [unhandled] [GET] http://localhost:3181/api/auth/verify?token=ffff…ffff
```

地址里那串 `token` 就是邮件正文里的明文凭据。这条路由（`server/api/auth/verify.get.ts`）当初就是为了**不让它出现在日志和响应里**才写成自己拼响应的——758 那一站讲的判据正是"查询串里带凭据就不许走 `throw`"。判据写在注释里，代码却漏了一条分支：只要数据库那一侧抛出错误，错误就一路冒到框架手里，而框架那一路要把完整地址写进日志、还要把完整地址放进响应体的 `url` 键。

## 占券那条语句在防什么

上一站把这句话整个读出来了（`[占券 SQL]`），三个条件都在 `where` 里：摘要相等、`used_at is null`、`expires_at` 严格大于传进来的时刻，改到行才算占到，`RETURNING` 只带回收件地址。

为什么不写成"先查一条没被用过的，再把它标记成已用"：那样两步之间有个窗口，两个人同时点开同一枚链接，两边都查到"没用过"，两边都登录成功。**一条带条件的 UPDATE 把"查、判、占"压成一次行操作**，数据库只让其中一个改到行。这是给并发用的，不是给人看的——代码读起来反而不如两步直白。

三种失败（不存在、已用过、已过期）走同一个分支、同一个 401、同一句人话、同一个 `LINK_INVALID`，`requestId` 照样在头与体里各一份。理由是：告诉探测者"这枚存在过但已被用过"，等于免费发送一枚有效链接的探测器。

## 这条 UPDATE 原子的是哪件事

它让**占券**这一步原子，它不让**这一次登录**原子。点开链接要落四批东西：占券、认人（不存在就建一个 free 账号）、把同址其它未用的券删掉、写会话行。占券成功之后认人失败，那张券已经写上 `used_at`——链接就此作废，用户只能重新申请一条。

这一站做的是把"失败"接住，不是把四步捆成一步：

```ts [server/api/auth/verify.get.ts]
let claimed: ClaimedToken | null
try {
  claimed = await claimLoginToken(token)
} catch (error) {
  return dbFailureResponse(event, error)
}

// 不存在、已用过、已过期：同一条路、同一个形状
if (!claimed) return apiFailureResponse(event, LINK_INVALID)

let userId: string
try {
  userId = await upsertUserByEmail(claimed.email)
  await revokeOtherTokens(claimed.email)
} catch (error) {
  return dbFailureResponse(event, error)
}
```

要不要跨语句的事务是一个真实取舍：捆起来，失败时链接还能再点一次；代价是行锁要跨过建用户那一步，还要把 `rotateSessionRow` 之外的事务写法请进这条路由。**本站先修泄漏，把"券白烧"留着**——它没有凭据外泄严重，而且修法会牵动服务层的调用形状。这条留在最后那节里。

## 判定顺序变成六档

| 什么情况 | 状态 | `data.code` | 走没走框架通道 |
| --- | --- | --- | --- |
| 没带凭据 | 400 | `TOKEN_MISSING` | 没走 |
| 形状不对（短串、非十六进制、参数重复） | 401 | `LINK_INVALID` | 没走 |
| 没配库 | 503 | `DB_NOT_CONFIGURED` | 没走 |
| 配了库但连不上、语句出错 | 503 | `DB_UNREACHABLE` | 没走（本轮新加） |
| 占不到券 | 401 | `LINK_INVALID` | 没走 |
| 会话行写失败 | 503 | `SESSION_CREATE_FAILED` | 没走 |

前三道闸门一个都不碰数据库，所以"库死了"不影响它们回什么——这也是垃圾输入不占用连接的那条断言的延续。

## 修之前与修之后

同一条命令（带一个连不上的连接串单独跑一趟，理由见下）：

```
[库连不上] {"状态":500,"url":"http://localhost:3181/api/auth/verify?token=ffff…ffff","框架指纹头":"DENY"}
[凭据回声] {"命中字段":["url"],"url":"http://localhost:3181/api/auth/verify?token=ffff…ffff"}
```

修之后：

```
[库连不上] {"状态":503,"code":"DB_UNREACHABLE","url":"/api/auth/verify","框架指纹头":"没有"}
[凭据回声] {"命中字段":[],"url":"/api/auth/verify"}
[闸门次序] {"没带凭据":[400,"TOKEN_MISSING"],"形状不对":[401,"LINK_INVALID"]}
```

`x-frame-options` 与 `content-security-policy` 是框架错误通道自己加的两位——它们不在，才证明这一条响应是自己拼的；`url` 只到路径，因为 `apiFailureResponse` 里填的是 `getRequestURL(event).pathname`。

这一组三条放在 `test/e2e/verify-db-down.spec.ts`，整份文件带一道开关：

```ts [test/e2e/verify-db-down.spec.ts]
test.skip(process.env.E2E_DB_DOWN !== '1', '需要 E2E_DB_DOWN=1 与一个连不上的 NUXT_DATABASE_URL 一起跑')
```

为什么必须单独一档：`verify.spec.ts` 那四条钉的是"**没配**库 ⇒ 503 `DB_NOT_CONFIGURED`"，悄悄给整趟带一个连接串，那条读数就换了意思（同一件事在 755 用 `NUXT_E2E_DB` 挡住过）。所以只有点名单跑这一份文件时，才把死连接串放进环境。

## 顺手纠正的一句注释

写第一版时我在注释里说"参数是分开传的，所以凭据不会跟着进日志"。这句话不对，是把两件事混成一件。为了确认，把日志里那个截断上限临时抬高重跑一趟，`reason` 掉出完整内容：

```
Failed query: update "login_tokens" set "used_at" = $1 where (… "expires_at" > $3) returning "email"
params: 2026-10-17T20:51:07.327Z,df0790f236013511e91fa4532fb7761f62320a51a3868dabf4a13fe5f53e3263,2026-10-17T20:51:07.327Z
```

中间那一位看着像凭据，它其实是这条语句的**参数**——摘要。算一下就知道：对 `'f'` 重复 64 次取 sha256，得到的正是 `df0790f2…3263`。而整趟输出里，`'f'` 那 64 个字符组成的串出现 0 次（同一趟里摘要出现 2 次，说明这趟扫描不是没读到东西）。

结论要换一种写法：**日志里会出现摘要，不会出现明文**，原因不是"参数不在 message 里"（它在），而是明文根本不是这条语句的参数，它在进数据库之前就已经被 sha256 过。截到 200 字只是控制行长，不是安全边界——把安全边界寄托在一句 `slice` 上是巧合，不是设计。

## 竞态这一条为什么不在这三份读数里

"两个人同时点同一枚链接，只有一个登录成功"是那句带条件 UPDATE 的存在理由，但它需要真库才测得出来：**假 db 不会替数据库做条件更新**（`test/unit/auth-verify.test.ts` 文件头原话），把手搓的假客户端换成 765 那份记录型 drizzle，看到的仍然是"两条请求都拿到行"，因为没人真的改行。读者的动手步骤是：连好自己的库、发一封真邮件，然后两个终端同时 curl 同一条链接——

```
curl -i "http://localhost:3000/api/auth/verify?token=<那一串>"
```

预期是一个 302 加两行 `Set-Cookie`、另一个 401 且 `data.code` 是 `LINK_INVALID`。

## 撞上这些症状的时候

| 症状 | 先看哪一位 | 为什么 |
| --- | --- | --- |
| 服务端日志里出现 `token=` | 有没有一条 `await` 在 `try` 外面 | 框架通道会把带查询串的完整地址打出来 |
| 响应体的 `url` 带参数 | 这条响应走的是框架通道还是自己拼 | 自己拼的那份只填 `pathname` |
| 稍后再点同一个链接报"无效" | 前面有没有失败发生在占券之后 | 占券已经写上 `used_at`，那次点击已作废 |
| 库里连不上却回 401 | 503 与 401 的分工 | 服务器没完成工作是 503，凭据不行才是 401 |
| 想看错误里的 SQL 只看到半句 | 日志的截断上限 | 参数被拼在 message 尾部，截断可能砍掉它 |

## 交付前自查

- 这条路由的每一个 `await` 都在能被接住的位置，没有一条能溜到框架手里
- 五种失败各自的 `code` 与状态码能对上 761 那张对照表，新加的码进了 `STATUS_TEXT` 的校验
- 明文凭据不出现在响应、日志与错误体里；摘要出现是已知且可解释的
- 需要真库的判据没有被假件冒充：那一条明确写成读者的动手步骤

没做的一件事，也是这一站留下的最具体的问题：**占券与认人之间没有跨语句的原子性**。库在认人那一步挂了的时候，链接白烧一张，用户要重新申请（还要再占一次令牌桶的额度）。修法是把这四步请进同一个事务，牵动服务层的调用形状，本站把它排在"多实例与规模"那一段之前处理。另外，发送失败与库失败要不要进计量表，本轮仍没量。

仓库这一轮的永久改动：`server/api/auth/verify.get.ts`（新增 `dbFailureResponse`，把占券、认人与同址撤券三段 `await` 都接住；文件头的判定顺序补上"库连不上"那一档，并写明参数会被拼进 message 这个事实）；新增 `test/e2e/verify-db-down.spec.ts`（三条，带 `E2E_DB_DOWN` 开关）。

## 本文依据

- 占券与认人：`server/services/auth/verify.ts`（`claimLoginToken`、`upsertUserByEmail`、`revokeOtherTokens`、`LINK_INVALID_CODE`）、`server/api/auth/verify.get.ts`
- 那条 UPDATE 的完整句子与三个参数：[765《登录链接全链（4）：券怎么生成：明文只活一瞬间，库里存摘要》](/posts/0df1bb5a78915330cdb18ad02deee4d1/) 里的 `[占券 SQL]`
- 本轮新读到的对照：`test/e2e/verify-db-down.spec.ts` 的 `[库连不上]`、`[凭据回声]`、`[闸门次序]` 三行原文
- 框架错误通道为什么必须绕开：`nitropack/dist/runtime/internal/error/prod.mjs:34-37`（那行 `[request error]`）与 `:56`（响应体的 `url`）；判据见 [758《接口错误契约（3）：throw 还是自己回响应，差别在查询串里有没有凭据》](/posts/bfa776071905ef16a29a234b510ddb65/)
- 503 的状态短语为什么要在表里：[759《接口错误契约（4）：状态短语只放 ASCII，表里没有的码也有兜底》](/posts/cbe7ef8c83f5341303324e49e39f01ea/)；新码由 `npm run check:errors` 校验
- "假 db 不替数据库做条件更新"：`test/unit/auth-verify.test.ts` 文件头
- 服务端路由与 `getQuery`：<https://nuxt.com/docs/4.x/guide/directory-structure/server>
- 上一篇（发信的三种失败与超时）：[766《登录链接全链（5）：发信那一步的两种失败：没配置与发不出去》](/posts/c29e1d95bff1fd152e3a6b60d18f170c/)
- 下一篇：[768《登录链接全链（7）：落地地址只认本站相对路径，登录成功先换号》](/posts/36c42da2c3340fcd9a4bf85ca7dc07da/)
- 本轮其它读数：这一组单独跑 `3 passed (26.5s)`；默认档里整份文件跳过，全量端到端跑出来是 `Running 192 tests using 10 workers`／`135 passed`／`57 skipped`／0 失败 (33.2s)；单位层仍是 27 文件／199 条；`npm run check` 八项全 ok，两份守卫（错误码与迁移）各打印 `问题 0`
