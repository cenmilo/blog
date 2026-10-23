---
url: /posts/9618f5556119d463bc99ead5317e820c/
title: 773 - 门后页与身份（4）：页面过了门不算过关，接口那道 requireUser 才是边界
date: 2026-10-25T12:00:00+08:00
updated: 2026-10-25T12:00:00+08:00
author: cmdragon

summary:
  摘要：会员区有两道闸：页面那层挡导航，接口那层挡数据。这一站把页面那道拆掉再跑一遍，看数据到底漏没漏；再把接口那层的三条规矩量成读数——先确认是谁在问、再验来源、最后才看参数，顺序颠倒会告诉没权限的人"什么样的编号是合法的"。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 接口鉴权
  - 错误码分工
  - 会员区
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 把页面那道门拆掉，再看数据漏没漏

前两站一直在说"门挡的是导航、不是数据"。这句话最直接的验证方式就是把它拆掉看看。把 `app/pages/console/index.vue` 第 5 到 7 行改成只留一个标题：

```vue
definePageMeta({
})
```

重跑会员区那批规格（`E2E_PORT=3229`，仍然没有数据库），打印出来的对照是这样的：

```
[console gate ssr] {"status":200,"location":"(没有这一位)","bytes":3077,"hasMarker":true,"hasRequestId":true}
[console gate followed] {"url":"http://localhost:3229/console","status":200,"text":true}
[console gate api] {"status":401,"code":"UNAUTHENTICATED"}
[metrics anon] {"status":401,"code":"UNAUTHENTICATED"}
[鉴权先于参数] {"非法形状":{"status":401,"code":"UNAUTHENTICATED"},"空编号":{"status":401,"code":"UNAUTHENTICATED"},"超长编号":{"status":401,"code":"UNAUTHENTICATED"}}
  2 failed / 4 skipped / 9 passed (27.9s)
```

红了的那两条就是钉门的判据，其余全部照绿——而这正是这一站要的结论，逐条对准（`[鉴权先于参数]` 那一行只截了状态码那半段，完整一行连同框架指纹头抄在后面那节）：

- 页面这层失守：状态从 302 变 200，正文从 89 字节涨到 3077 字节，`hasMarker` 从 `false` 变 `true`，那句"这一段只有登录用户才应该看得见"确实发给了一个匿名访客。
- 数据一位都没漏：会员区仪表盘要用的那个接口仍然 401，`code` 仍是 `UNAUTHENTICATED`；后台那条改会话的接口对三种非法形状仍然是 401。它们判的是 `event.context.user`，跟页面渲了什么无关。
- `text: true` 说明这一页对匿名访客显示的还是"未登录"——因为身份那一位（上一站量的负载）仍然是 `null`。拆掉门泄露的是**页面骨架与文案**，不是别人的数字。

所以"这道门要不要修"的答案是：要修，但它的严重级别和"接口没守卫"完全不是一档。讲会话的那一节配方把这句话写得很直白：「Client-side middleware is helpful for the user, but without server-side protection your data can still be accessed.」

改回原样再跑同一批规格（`E2E_PORT=3230`）：`8 passed`／`4 skipped (26.9s)`。

## 接口这一层的三条规矩

`requireUser` 的函数体很短，但它站着的三条规矩都在注释里，本轮把它们变成判据。

**中间件不拦人。** 讲 server 目录的那一节写着：「Middleware handlers should not return anything (nor close or respond to the request) and only inspect or extend the request context or throw an error.」所以 `server/middleware/auth.ts` 只做一件事：把这个人挂到 `event.context.user` 上，或者挂 `null`。拦不拦是每个接口自己的决定——这句话在 770 那站是背景，在这一站是判据。

**没登录与没权限是两个码。** `requireAdmin` 的第一行就是 `requireUser(event)`：没人就 401，有人但角色不对才 403 `FORBIDDEN`。把它们混成一个 401 的写法很常见，代价落在客户端：分不清"该去登录"还是"登录了也进不去"，只能把 403 也当未登录处理，跳去登录页再回来一次。

**守卫不查库。** 两份守卫读的都是 `event.context.user`，那是中间件那一次查询的结果；一个接口里调三次守卫也还是一次查询。这条不是风格：查询次数是 770 量过的读数，多一处查库就多一处能被人拿时间差观察的地方。

单位层那两份用例（`require-user.test.ts` 六条、`require-admin.test.ts` 五条）钉的形状是：没登录抛 401、`data.code` 是 `UNAUTHENTICATED`，登录了但角色不对抛 403、`data.code` 是 `FORBIDDEN`；两者的 message 都不含 `sid`／`cookie`／`sess_` 这些字样，也不点名是谁；错误对象序列化出来的键固定是 `message`、`statusCode`、`statusMessage`、`data` 四个。

**这一段说的是单位层量到的形状**，最后那四个键尤其要说清范围：它是错误对象自己序列化的键，不等于浏览器收到的响应体——响应体由框架再包一层，下一节把它原样量出来了（多了 `error` 与 `url` 两位）。

## 顺序才是这一站的干货

后台那条"全端下线"的接口里，四步的次序是有意的：**先确认是谁在问（401），再验动作来源（403 `CSRF_MISMATCH`），最后才看参数（400），然后才取数。**

反过来写会发生什么，本轮量了。把参数校验整段挪到守卫之前，再打那三种非法形状：

```
[鉴权先于参数] {"非法形状":{"status":400,"code":"USER_ID_INVALID"},"空编号":{"status":400,"code":"USER_ID_INVALID"},"超长编号":{"status":400,"code":"USER_ID_INVALID"}}
  Expected: 401 / Received: 400，2 failed
```

（这一行打印还有后半段——框架指纹那三位与"回声出现在哪一位"，原样抄在下一节，这里只截状态码那半段。）

改动只有一处挪位置，症状是：**一个连身份都没证明的人，拿到了"这个编号形状不合法"这句反馈。** 单看一条没什么，能反复问就等于在描摹内部校验的形状——什么长度的编号会被拒、空值走哪一路。这类信息不是秘密，但它是免费的探测面。

同一处改动在静态判据那一侧的读数更早：`[鉴权顺序]` 里那一条变成 `{"守卫行":26,"读参数行":20,"首个await行":32,"顺序对":false}`，`expected false to be true`。**行号能比出来的东西，不用等一次构建。**

## 401 走的是框架通道，地址就在响应里

这一条是本轮的意外收获。`requireUser` 抛的是 `createError`，所以这一路的响应由框架组装，形状是六键：

```
[两个接口的 401] {"data形状数":1,"data的键":["code","requestId"],
  "顶层键":["error","url","statusCode","statusMessage","message","data"],"顶层带url":true,
  "两个url":["http://localhost:3230/api/console/metrics","http://localhost:3230/api/admin/users/not-a-uuid/sessions"],
  "会话号出现":false}
```

两个不同接口的 401，`data` 那一段完全同形（客户端按 `code` 分支就能覆盖全站），但顶层多出来的 `url` 那一位**装的是完整地址**。于是路径里那一段用户编号被原样回声了一次——同一行打印的完整样子：

```
[鉴权先于参数] {"非法形状":{"status":401,"code":"UNAUTHENTICATED"},"空编号":{"status":401,"code":"UNAUTHENTICATED"},"超长编号":{"status":401,"code":"UNAUTHENTICATED"},"框架指纹头":{"x-frame-options":"DENY","content-security-policy":"script-src 'none'; frame-ancestors 'none';","x-request-id":"带号"},"回声出现在哪一位":["url"],"message里有没有回声":false}
```

三位一起看：**回声出现在 `url`，不在 `message`**；而 `x-frame-options` 与那条 CSP 都在，说明这一路走的是框架的错误通道。这条接口没有凭据在地址里，回声回的是调用者自己填的东西，无害。但登录链接那条路由不一样——它的地址里带着一次性凭据，所以 767 把它整条改成自己组装响应，那时量的指纹恰恰是"框架给错误响应加的那几位安全头不在"。**同一个站里，"走框架通道"和"自己回响应"是两种形状**，判据要按地址里有没有凭据来选，不能全站统一。

## 接线判据：门后目录里不许有裸接口

`server/api/console/**` 与 `server/api/admin/**` 这两个目录名本身就是承诺。基线打印：

```
[门后接口] [{"接口":"server/api/console/metrics.get.ts","守卫":18},{"接口":"server/api/admin/users/[userId]/sessions.delete.ts","守卫":20}]
[认证流程] [{"接口":"…/github/callback.get.ts","守卫行":-1},{"接口":"…/github/start.get.ts","守卫行":-1},{"接口":"…/login-link.post.ts","守卫行":-1},{"接口":"…/logout.post.ts","守卫行":-1},{"接口":"…/verify.get.ts","守卫行":-1}]
[守卫自身] [{"文件":"server/utils/auth/require-user.ts","读context":true,"委托给requireUser":false,"查库":false,"有await":false},
  {"文件":"server/utils/auth/require-admin.ts","读context":false,"委托给requireUser":true,"查库":false,"有await":false}]
```

五条规矩：门后目录每个接口都得挂守卫；`admin` 那侧必须是 `requireAdmin` 而不是只 `requireUser`；守卫必须先于"读请求里的东西"和任何一次取数；认证流程那几支**不许**挂守卫（没登录的人要能申请登录链接，挂了就是把门焊死在正确的方向上）；两份守卫自己不查库、不 `await`，其中 `requireAdmin` 必须把"是谁"这一步委托给 `requireUser`。

三条变异读数：

- 参数校验挪到守卫前 → 静态 1 红、端到端 2 红（见上一节两组读数）。
- 后台那条只挂 `requireUser` → `[后台那道] {"接口":"…/sessions.delete.ts","有admin":false}`，1 红。这一种的后果是任何登录用户都能把别人的会话全删掉，而页面层完全看不出异常。
- 给登录链接入口加一句 `requireUser(event)` → `[认证流程]` 里那一条变成 `守卫行:11`，1 红。

这份判据自己也被纠正过两次，都留着：第一次跑假定两份守卫都会读 `event.context.user`，而 `require-admin.ts` 不自己读（它委托），所以那条断言改成"一个读、一个必须委托"；第二次是类型检查报 `test/unit/api-guard-surface.test.ts(140,9): error TS2783: '文件' is specified more than once, so this usage will be overwritten.`——对象里先写了 `文件`、后面又展开了一份带同名键的对象，后者会盖掉前者。这条线连续三站在同一件事上被自己写的检查打回：**跑起来了不等于类型也过**。

## 出问题时按这个顺序找

| 症状 | 大概率是哪一位 | 怎么确认 |
| --- | --- | --- |
| 匿名能看到门后页正文，但数字是空的 | 页面那层门没了 | 看接口是不是仍 401；再看那一页有没有声明门 |
| 没登录的人拿到 400 | 鉴权与参数校验的顺序倒了 | `[鉴权顺序]` 的行号对照 |
| 登录页跳来跳去回不来 | 403 被当成 401 处理 | 看那一路的 `code` 是不是 `FORBIDDEN` |
| 一次页面访问查了两次会话表 | 有人在接口里自己查库 | `[守卫自身]` 的 `查库` 与 `有await` |
| 登录链接接口回 401 | 认证入口被挂了守卫 | `[认证流程]` 里那一条 `守卫行` 不是 -1 |

## 你自己站上跑一遍

```
npm run test:unit -- test/unit/api-guard-surface.test.ts
E2E_PORT=3230 npm run test:e2e -- --project=site \
  test/e2e/api-guard.spec.ts test/e2e/console-gate.spec.ts test/e2e/metrics-endpoint.spec.ts
```

预期 `Tests 5 passed (5)` 与 `8 passed`／`4 skipped`。**403 这一路本站量不到**：要一个真存在的会话行、还得是 `free` 角色，才能看出"登录了但没权限"那道 403。这一段是留给读者的待跑步骤，用你自己的库跑（连接串放 `.env`，别写进会被渲染或提交的文件），单位层那五条先钉住形状。

本轮之后单位层是 34 文件／232 条，运行时层 6 文件／20 条。永久改动是两份判据文件（`test/e2e/api-guard.spec.ts`、`test/unit/api-guard-surface.test.ts`）；四处变异与两处判据自身的错全部还原，`server/api/admin/users/[userId]/sessions.delete.ts`、`app/pages/console/index.vue` 都是原样。

## 本文依据

- 讲会话的那一节配方（客户端中间件对用户有用、但服务端不保护数据照样能被取走；未登录要在接口上回 401）：<https://nuxt.com/docs/4.x/guide/recipes/sessions-and-authentication>
- 讲 server 目录的那一节（服务端中间件不该返回任何东西，只能检查或扩展请求上下文、或者抛错）：<https://nuxt.com/docs/4.x/directory-structure/server>
- 这一站读过的四个文件：`server/utils/auth/require-user.ts`、`server/utils/auth/require-admin.ts`、`server/api/console/metrics.get.ts`、`server/api/admin/users/[userId]/sessions.delete.ts`
- 读数出处：`test/e2e/api-guard.spec.ts`（本轮新增）、`test/unit/api-guard-surface.test.ts`（本轮新增）、`console-gate.spec.ts`、`metrics-endpoint.spec.ts`、`require-user.test.ts`、`require-admin.test.ts`
- 上一篇：[772《门后页与身份（3）：门后页的 HTML 写着谁，匿名访客看到的又是什么》](/posts/d77d34784ab45a75a96762f1c39dbe0f/)
- 下一篇：[774《门后页与身份（5）：登出要删对 path 与域，全端下线是另一件事》](/posts/c8b85fcb6706255caf501ce07f3d2b4a/)
