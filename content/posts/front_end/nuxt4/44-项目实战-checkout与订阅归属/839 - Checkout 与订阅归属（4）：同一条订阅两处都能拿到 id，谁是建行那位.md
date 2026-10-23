---
url: /posts/4cdb53d89d2635073d08506cd7c0c438/
title: 839 - Checkout 与订阅归属（4）：同一条订阅两处都能拿到 id，谁是建行那位
date: 2026-11-05T12:00:00+08:00
updated: 2026-11-05T12:00:00+08:00
author: cmdragon

summary:
  摘要：订阅那串 `sub_…` 有两个地方能读到——下单时渠道回给你，事件回来时载荷里也带着。这一站把"到底哪一路把库里那一行建出来"问清楚：下单那一路连订阅表都没引进来，入账单子上只有两条规则能作用在"还没有过订阅"的起点上。顺着这个问题往下，读出渠道送回来的用户编号本站不认得时发生的两件事：库不报"查不到"，报"这串不是 uuid"；而半途崩掉的那条事件，重投进来会被当成重复事件直接吞掉。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 订阅归属
  - 外键
  - webhook
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先看见的是终端里那两行报错

上一站结束时，`server/api/billing/checkout.post.ts` 交出去的那张单子里，`metadata[userId]` 已经写着本次会话主人的 uuid，事件回来时入账那一路读的正是这一格。把一个本站不认得的编号塞进这一格，服务端日志里躺着两种报错：

```text
H3Error: Failed query: select "id" from "users" where "users"."id" = $1 limit $2
  cause: error: invalid input syntax for type uuid: "00000000-0000-4000-8000-452002887"
  statusCode: 500,
```

```text
H3Error: Failed query: insert into "subscriptions" ("id", "user_id", "customer_id", "status", ...) values (...)
  cause: error: insert or update on table "subscriptions" violates foreign key constraint
          "subscriptions_user_id_users_id_fk"
  statusCode: 500,
```

第一条连库都没查到东西，是那条 `select` 自己炸的；第二条是外键拦的。两发的回调都回 500，而渠道对非 2xx 会重试，重试时会换一个新的时间戳重新签一次——签名照样过，于是这条事件会一遍一遍回来。

这是这一站要修的。修之前先把标题那个问题问清楚，因为它决定了要在哪一层修。

## 订阅 id 有两个来源，建行只有一个

`subscriptions.id` 那一列用的就是渠道给的那串 `sub_…`（830 定的取舍）。两个地方都能读到它：下单那一路（创建会话的响应里）与事件回来那一路（载荷的 `data.object.id` 或 `subscription`）。两处都建行的话，两处都得处理"这一行已经有了怎么办"，而下单那一路建出来的行连状态都填不出来——那时候渠道只给了一个跳转地址。

本站的做法是下单这一路一个字都不写订阅表。这条不是推测，两处各判了一次。

动态那一判把请求真的打到放行（渠道根地址指本机一个假 provider，它只替本站记下送出去的正文）：

```text
[结账·不写订阅表] {"放行状态":200,"渠道回了地址":"http://127.0.0.1:3411/c/cs_local_836",
"假provider替本站记下几发":1,"下单前那一位的订阅行数":0,"下单后那一位的订阅行数":0}
```

`假provider替本站记下几发` 必须是 1，不然"库里没行"只是"这一路根本没跑到底"。

静态那一判更硬，它不看行数、看这一路有没有可能写库：接口与服务层两个文件里，`useDb(`、`from(subscriptions)`、`insert(subscriptions)` 一个都不出现，连表定义都没引进来。

```text
[结账·不碰库] {"接口碰库":false,"服务碰库":false,"两处引入表定义":false}
```

于是建行的只有入账单子。它也**不是来一条建一条**：那五格事件里，能作用在"还没有过订阅"这个起点上的规则只有两条——结账完成建成生效，订阅没了建成终态。变更事件与两类发票事件打在一个不存在的订阅上，规则表里对不上任何一行。

```text
[入账·谁建行] {"变更":{"状态":200,"行数":0,"事件行":{"在":true,"类型":"customer.subscription.updated","处理过":true}},
"变更开关":{"状态":200,"行数":0"},
"结账":{"状态":200,"行数":1,"状态位":"active","归属":"c2923622-0d98-4caa-b935-b208c171f8ec"},
"删除":{"状态":200,"行数":1,"状态位":"inactive"}}
```

四条打印值对上四条结论：变更事件先来什么都不会建，但它**照留痕、照标记处理完**（留痕与行动是两件事）；结账事件后到才把那一行建起来，归属就是载荷里那一位；"订阅没了"先到会建出一行 `inactive`。

最后这一条值得停一下。这行记录是本站从没见过的一个订阅，而且它一进来就是终态——状态机里终态之后任何事件都不复活。留着它是"渠道确实发过这个东西"的历史，代价是订阅表里多了一行永远不会动的记录。

## 库不跟你说"查不到"，它跟你说"这串不是 uuid"

`users.id` 是 uuid 那一列。PostgreSQL 的写法是**八位、三位四位、再加十二位，一共三十二个十六进制字符**，输入时少一位多一位都不认。而 `metadata.userId` 是渠道替我们保管的**任意文本**：它来自别的一套系统、来自一个已经删掉的账号、或者来自别人把测试环境的密钥配到了本站的 endpoint。

所以两道挡的位置不一样。形状这一道必须在问库**之前**，否则那条查询自己就抛；存在这一道只能问完库才知道，这一步躲不掉。都在 `server/services/billing/ingest.ts` 里，只在"要新建那一行"这条路上跑：

```ts
const 本站用户id形 = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i

if (!本站用户id形.test(认出.用户id)) {
  await 处理完()
  return { 抢到新事件行: true, 做了什么: '归属那位形状不对' }
}
const 有人 = await 库.select({ id: users.id }).from(users).where(eq(users.id, 认出.用户id)).limit(1)
if (有人.length === 0) {
  await 处理完()
  return { 抢到新事件行: true, 做了什么: '归属那位查无此人' }
}
```

`已有行的更新`那一路不动归属，所以每次回调并不多这一趟查询——这一位也被静态档钉住（更新那段里既不写 `userId` 也不查 `users`）。

认不到怎么办？照"读不懂"那一族处理：事件行留着、`processed_at` 盖上、**不建订阅行**、回 200。两种挡各判一档，都读到同一组形状：

```text
[入账·归属形状] {"塞进去的那一位":"00000000-0000-4000-8000-452295477","响应":{"状态":200,"码":"(体里没有这一位)"},
"事件行":{"在":true,"类型":"checkout.session.completed","处理过":true},"订阅行":{"行数":0,"状态":null,"时间毫秒":null,"归属":null}}
[入账·查无此人] {"响应":{"状态":200,"码":"(体里没有这一位)"},
"事件行":{"在":true,"类型":"checkout.session.completed","处理过":true},"订阅行":{"行数":0,"状态":null,"时间毫秒":null,"归属":null},
"本站真人那头的行数":0}
```

最后那一位是"别把行建到别人名下"的证据：一个形状合法却不存在于本库的 uuid，如果代码拿它去做了别的事，最坏的写法不是建不出行，而是建到某个真用户头上。

入账动作那份清单从九种变十种，两种挡各占一个字。调用方分支靠这一位，不靠猜：

```text
[入账·动作清单] {"成员":["重复事件","已迁状态","状态没变","没订阅行可改","缺新建订阅行所需的信息",
"归属那位形状不对","归属那位查无此人","渠道状态不认得","载荷认不出","没库"],"种数":10,"重复的":[]}
```

## 崩在中途的那一行，重投救不回来

上面两种报错还有一个副产品，比 500 本身更值得记。入账的顺序是**先占位再处理**：那句带唯一约束的 `insert … on conflict do nothing … returning` 一落下来，这条事件就"见过"了。所以任何让后面那几步中断的错误，都留下一行 `processed_at` 为空的事件行；而渠道重试送来的同一条撞到的正是同一个主键 ⇒ 拿不到返回行 ⇒ 判成"重复事件"，直接回 200，**状态机一次都不跑**。

这条不用制造异常也能读：直接落一行空标记位的事件，再把那条本该建行的事件按原样发进去。

```text
[入账·空行被吞] {"空行在前":{"在":true,"类型":"checkout.session.completed","处理过":false},
"重投":{"状态":200,"体":{"received":true}},"空行在后":{"在":true,"类型":"checkout.session.completed","处理过":false},
"订阅行":{"行数":0,"状态":null,"时间毫秒":null,"归属":null}}
```

回的是 200，库里什么都没有，那一格的标记位仍然是空。本站现在**没有**任何扫空标记位的补处理任务，这条挂在未闭合里；判据把"重投也不会补跑"钉成一条用例，将来谁加了补处理，这一条会红，改判据的人就知道产品行为变了。

## 三次改动，各红了哪几档

判据能不能证伪只有靠改坏代码来验。三次各改一处，红的档数正好指出改动落在哪一层。

| 改坏的那一处 | 端到端 | 单测（静态） |
| --- | --- | --- |
| 把形状挡去掉 | `839⑨` 一档红：状态 500、标记位 false | — |
| 把"查不到就不建行"改成"照样往下写" | `839⑩` 一档红：状态 500，报的是外键那句 | — |
| 多写一次认人查询 | — | `⑤` 红：`expected 2 to be 1` |
| 在结账服务里引进表定义 | — | `⑬` 红：`两处引入表定义:true` |

前两次各红一档，说明两道挡是彼此独立的：只留形状挡，一个合法而不存在的 uuid 照样撞外键；只留认人挡，一条语法错的输入在那次查询里就炸了。后两次红的是静态档，它们不需要库——这也是为什么这一层的顺序判据值得单独写一遍：把认人查询挪到插入之后，带库那几条端到端不一定立刻看得出，而静态那条第一趟就红。

## 一个数错了的读数：全表计数在并行里串味

`839⑩` 第一版除了订阅行数还数了一次全表的事件行数，单跑绿：

```text
{"下单前":{"订阅行数":0,"事件行数":9},"下单后":{"订阅行数":0,"事件行数":9}}
```

把这一档放回整套端到端并跑，同一句判据红了：`Error: 下单这一路往事件表里写了东西`。隔壁那些用例正在写自己的事件行，这个 9 是全站的累计值，跟本次请求没有关系。这与 834 那一站撞的是同一个坑——想问"这一行动没动"要读行内的东西，想数"这一位有几行"要按本次探针筛，**任何全表计数在并行判据里都会串味**。改完的写法只数属于本次探针那一位的订阅行，"没有写别处"这件事交给静态档去钉。

## 跑一遍

命令里两个密钥与库地址都换成你自己的值，端口每次换一个没用过的（这台机器 36xx 段常有出站连接占位，报 `listen EADDRINUSE` 而查不到监听进程时就是撞上出站连接了）。

```bash
# 入账那一路（要库＋签名密钥，十一条全跑；不带这两个变量十一条全跳）
NUXT_E2E_DB=1 NUXT_STRIPE_WEBHOOK_SECRET=whsec_你的本地密钥 E2E_PORT=3641 \
  npm run test:e2e -- test/e2e/webhook-idempotency.spec.ts

# 下单那一路（要库＋下单密钥＋渠道根地址指本机假 provider）
NUXT_E2E_DB=1 NUXT_STRIPE_SECRET_KEY=sk_test_你的本地密钥 \
  NUXT_STRIPE_API_URL=http://127.0.0.1:3411 NUXT_STRIPE_WEBHOOK_SECRET=whsec_你的本地密钥 E2E_PORT=3642 \
  npm run test:e2e -- test/e2e/billing-checkout-gate.spec.ts

# 静态档与整套检查（不需要库）
npm run test:unit -- billing-ingest billing-checkout
npm run check
```

本轮读到的计数：单测 **335 条／51 个文件**；八项检查全过（`check:errors`、`check:migrations`、`lint`、`typecheck`、`test:unit`、`build`、`test:nuxt` 逐项 ok，尾行「全部通过」）；两份端到端与另三份相关档并跑（带库那一趟）**29 passed／4 skipped**；整套端到端不带库那一趟 **257 passed／107 skipped**，跳过的 107 条按原因归成四类（要库 47、要外部服务变量 35、要内容目录读得到 14、要另一趟构建的开关 11），没有一条归不进类。

## 撞上的四件事

| 症状 | 为什么 | 怎么办 |
| --- | --- | --- |
| 回调 500，库里那一行 `processed_at` 是空的 | 占位已经落下、处理中断 | 把会让处理中断的输入在写库之前挡掉；已经空着的那些，本机没有补处理，重投也救不回来 |
| `invalid input syntax for type uuid` | 拿任意文本去比 uuid 那一列，库报语法错而不是"查不到" | 问库之前先按 `8-4-4-4-12` 判形状 |
| `Expected 0 arguments, but got 1` | 测试里那个本地箭头函数不收参数，调用时却递了客户端进去 | 闭包里已经握着那个连接，调用别带参数 |
| 单跑绿、并跑红的"表级计数"判据 | 并行用例同时在写别的行 | 按本次探针筛（`where user_id = $1`），或者换成静态档 |

## 下一站

付款之后会员区那一栏要显示什么。前面这些都写在库里的表上，而页面第一次要真去读 `subscriptions` 那两列——`status` 与 `current_period_end`。那时"这一行是谁建的"就不再只是入账的问题：读的一方对 `inactive`、对空着的周期末、对刚从逾期回到生效的那一行，各要给出什么。

## 本文依据

本站这一侧：

- 事件入账（两道归属挡、`processed_at` 的语义）：`server/services/billing/ingest.ts`
- 入账动作清单与静态顺序判据：`test/unit/billing-ingest.test.ts`
- 下单那一路六道判断的顺序与"不写订阅表"：`server/api/billing/checkout.post.ts`、`test/unit/billing-checkout.test.ts`
- 谁建行的端到端读数：`test/e2e/webhook-idempotency.spec.ts`（`839⑧⑨⑩⑪`）、`test/e2e/billing-checkout-gate.spec.ts`（`839⑩`）
- 服务端目录与测试分层：https://nuxt.com/docs/4.x/guide/directory-structure/server 、https://nuxt.com/docs/4.x/getting-started/testing

渠道与库那一侧（字段与格式以之为准，正文本轮取过）：

- UUID 的写法与总位数：https://www.postgresql.org/docs/current/datatype-uuid.html
- 订阅对象（`status` 的取值、`customer.subscription.deleted` 的形状）：https://docs.stripe.com/api/subscriptions/object
- 事件对象（`data.object`、`created` 单位是秒）：https://docs.stripe.com/api/events/object
- 非 2xx 会重试并重新签名：https://docs.stripe.com/webhooks
