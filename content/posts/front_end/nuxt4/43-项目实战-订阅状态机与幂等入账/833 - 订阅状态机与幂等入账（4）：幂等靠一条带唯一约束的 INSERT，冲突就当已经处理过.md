---
url: /posts/d5deaec8e5f2630a2922c14b9840547c/
title: 833 - 订阅状态机与幂等入账（4）：幂等靠一条带唯一约束的 INSERT，冲突就当已经处理过
date: 2026-11-04T12:00:00+08:00
updated: 2026-11-04T12:00:00+08:00
author: cmdragon

summary:
  摘要：同一条事件送来两次，库里就该只有一行、状态只迁一次。这件事不能靠应用记一个见过的 id 集合，进程重启就忘了。这一站把幂等写成一句带唯一约束的插入：撞了主键就当已经处理过，直接回 200，状态机一次都不跑。顺手定下事件表那一列标记位的含义，并用库里的统计视图读出一件数据面看不见的事——白写一次。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 幂等
  - webhook
  - PostgreSQL
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 同一条事件送来两次，库里不该多出第二行

上一站把门装好了：签名不对的进不来。这一站处理进来的那些。渠道会重发——你回非 2xx 它就再投一次。

最省事的写法是在内存里记一个"处理过哪些 id"的集合。它在两个地方失效：进程重启就清空，多实例各记各的。幂等必须落在**数据库的唯一约束**上，只有它跨进程、跨重启还认账。上一站建的两张表正好为这件事准备：`stripe_events` 的主键是渠道给的事件 id（`evt_…`），`subscriptions` 的主键是渠道给的订阅 id（`sub_…`）。别人给的 id 天然唯一，拿来当主键，重放就被一句 INSERT 挡下来。

## 顺序比语句本身更容易写错

入账那一步在 `server/services/billing/ingest.ts`，只有三拍：

```ts
const 抢到 = await 库
  .insert(stripeEvents)
  .values({ id: 事件id, type: 事件类型 })
  .onConflictDoNothing()
  .returning({ id: stripeEvents.id })

if (抢到.length === 0) {
  return { 抢到新事件行: false, 做了什么: '重复事件' }
}
// 抢到之后才读现状、跑状态机、按"改没改"决定写不写订阅行
```

写成"先 `select` 看有没有、没有再 `insert`"也照样能跑，但那是两条语句，中间有窗口：并发投同一条时两边都读到"没有"，于是都插、都跑一遍状态机。`onConflictDoNothing` 把判断交给库，`returning` 把"到底是不是我插进去的"读回来。这与登录链接那一站"占券为什么是一条带条件的 UPDATE"同族——**冲突不是错误，是信息；返回的行才是谁抢到**。

端到端判不了顺序：把状态机挪到插入之前，结果一位不变、判据照样全绿。所以有一档静态判据读源码，只问三件事：占位那句是否连在一起、有没有先查再插、"抢不到就返回"是否排在调用状态机之前：

```json
[入账·占位] {"段":"insert(stripeEvents)\n    .values({ id: 事件id, type: 事件类型 })\n    .onConflictDoNothing()\n    .returning","带冲突不处理":true,"要返回行":true,"撞主键前先查过":false}
[入账·顺序] {"抢不到的位置":2073,"状态机的位置":2757,"顺序对":true}
```

## 重放那一次，库里一位都没动

七条端到端判据都要库也要签名密钥；不带那两个变量的那一趟七条全 skip。第一次进来读到的是"事件行落下、订阅行建起来、标记位盖上"；同一条字节原样再发一次：

```json
[入账·重放] {"第一次":{"状态":200},"重放":{"状态":200,"体":{"received":true}},"重放前的订阅行":{"行数":1,"状态":"active","时间毫秒":1792440002000},"重放后的订阅行":{"行数":1,"状态":"active","时间毫秒":1792440002000},"事件行数":{"在":true,"类型":"checkout.session.completed","处理过":true}}
```

三件事同时成立才算数：重放回 200（回 409 或 500，渠道就把同一条一直重试下去）、订阅行的时间毫秒**逐位相同**（动了说明状态机被重跑）、事件行还在。

## 那一列可空的标记位，含义在这一站定下来

`stripe_events` 有两列时间：`received_at` 非空、`processed_at` 可空。这一站给它一句话的定义——**空＝还没处理完；非空＝本站已经处理完这条事件，包括"看完决定不行动"那一种**。它不表示"订阅行被改过"，所以三种没动订阅的情况都要盖上这一位：类型不认识、载荷读不懂、渠道报了一个我们词表里没有的状态名。

```json
[入账·读不懂] {"陌生类型":{"状态":200,"行":{"在":true,"类型":"payout.paid","处理过":true}},"半条带类型":{"状态":400,"码":"WEBHOOK_SIGNATURE_INVALID"},"半条不换类型":{"状态":200,"码":"(体里没有这一位)"},"兜底行数变化":{"前":0,"后":1},"最近兜底行":{"id":"unparsed-a880cc21-…","type":"未知","处理过":true},"订阅表这一档的行数":0}
[入账·词表过期] {"建好":{"行数":1,"状态":"active","时间毫秒":1792439114000},"未登记":{"状态":200},"之后":{"行数":1,"状态":"active","时间毫秒":1792439114000},"事件行":{"在":true,"类型":"customer.subscription.updated","处理过":true},"没动":true}
```

读不懂也回 200 是有决定的：签名真过说明字节确实来自渠道，只是本站读不懂；打成 4xx 它会重试并重新生成签名，一条读不懂的事件就在队列里出不去。于是收下、留痕、盖标记位、不动订阅行。

另一件事：读不懂时**没有可信的事件 id**。第一版写空串，症状是第二条读不懂的载荷撞上那行空主键、被当成"重复事件"处理掉。现在用本站请求号兜（`unparsed-<requestId>`），留痕有但不冒充渠道的 id。"类型不认识"则是另一回事——顶层那个 `evt_…` 照样读得出来，主键就得继续用它，否则重投时认不出来。

## 渠道的八个状态名，站内只有五个位置

上一站的状态机只吃站内语义，渠道那八个取值里只重合 `active` 与 `past_due`。翻译放在 `server/services/billing/map.ts`，每格写理由：

| 渠道那格 | 站内落到 | 凭什么 |
| --- | --- | --- |
| `active` | `active` | 两边同一个名字（重合的两格之一） |
| `past_due` | `past_due` | 两边同一个名字（另一格） |
| `trialing` | `active` | 试用期内权益可用，站内那一格只问能不能用 |
| `incomplete` | `none` | 首笔还没付成，权益没开始 |
| `unpaid` | `past_due` | 逾期未付，站内最接近的是逾期那一格 |
| `incomplete_expired` | `inactive` | 首笔没付成而单子已过期，不会再自己活过来 |
| `canceled` | `inactive` | 已取消，终态 |
| `paused` | `inactive` | 暂停期间权益不发（这一格有风险，见文末） |

这张表是**本站的解释**，期望值全部手敲；再加两条对账：清单登记了但 `switch` 漏一格要能红，翻出站内清单之外的名字也要能红。

```json
[映射·对账] {"清单数":8,"翻不出":[],"站内名集合":["none","inactive","active","past_due"],"越界":[],"站内清单数":5}
[映射·发票词表] {"发票清单":["paid","open","uncollectible","void"],"挡住的":["paid:挡了","open:挡了","uncollectible:挡了","void:挡了"],"没挡住":[]}
```

`站内名集合` 只有四个：`cancellation_scheduled` 不由渠道名字产生，是站内从 `cancel_at_period_end` 推出来的。

**发票的 `status` 是另一套词表**，而 `invoice.paid` 与 `invoice.payment_failed` 的 `data.object` 恰好就是发票；混用的后果是把发票状态写进订阅表。三类对象也都有 `id`，但订阅 id 只在 `object === 'subscription'` 时从 `id` 取，发票与会话要从 `subscription` 那一格取——拿发票的 `in_1` 当订阅 id，库里就多出一行谁也不认识的订阅。

渠道报一个没登记的名字时，本站取**宁可不迁**：状态不动、回 200、事件照留痕。打成 500 会让渠道的升级窗口里所有事件都在重试。

## 只有库里看得见的那件事：白写一次

`applySubscriptionEvent` 返回 `changed` 是上一站定的，写库这一步要不要听它？把那句判断短路掉跑一次，七条判据只红一条，而且不是"数据错了"那条：

```json
[入账·乱序] {"新事件":{"状态":200,"订阅行":{"行数":1,"状态":"cancellation_scheduled","时间毫秒":1792440005000}},"旧事件":{"状态":200,"订阅行":{"行数":1,"状态":"cancellation_scheduled","时间毫秒":1792440005000}},"两条事件都在":{"新":true,"旧":true},"时间戳没动":true,"这条旧事件引起的更新次数":1}
```

"旧事件不许改回新状态"照样是绿的——状态机没改结论时返回的就是同一个现状，写回去的值与原值相同。数据看不出差别，只能问库里的统计视图：

```ts
await client.query('select pg_stat_force_next_flush()')
const 结果 = await client.query(
  "select n_tup_upd from pg_stat_user_tables where relname = 'subscriptions'",
)
```

还原之后这一位读回 **0**，红的就是那条判据自己那句话：旧事件白写了一次（状态机说了不改，写库那一步没听）。白写不是无害的——挂在更新上的东西（触发器、审计、复制流量）都跟着空转一遍。

## 缺归属就别写半行的订阅

`subscriptions` 有三列非空：`user_id`、`customer_id`、`price_id`。本站的身份放在 `metadata.userId`，那是创建订阅时写进去的——渠道不认识本站的用户 id。缺任何一位就一行都不写，而不是补占位值：半空的订阅行后面每一站都要加特判。

```json
[入账·缺归属] {"响应":{"状态":200,"体":{"received":true}},"事件行":{"在":true,"类型":"checkout.session.completed","处理过":true},"订阅行":{"行数":0,"状态":null,"时间毫秒":null,"归属":null}}
[入账·删用户后事件行] {"在":true,"类型":"checkout.session.completed","处理过":true}
```

第二行把上一站那条设计读实：`stripe_events` 没有外键，删掉用户之后那一行仍在。**订阅行跟着账号走，渠道发过什么不跟着走**。

## 探针对自己也会改字节

这一站最费时间的不在代码里，在判据里。想发一条"签名真过但载荷不是合法 JSON"的字节，就把半条 JSON 带上 `content-type: application/json` 发出去。第一跑拿到 400，于是正文写下"框架会先把没签名的请求挡在门外"。

这句话是错的：结论来自一次没配对的观察。同一个字符串只把 `content-type` 换成 `text/plain` 再发，回 200。差在哪？临时在服务端打一位长度：

```text
[WebServer] {"msg":"probe833","原文长度":32,"头在":true}   ← 带 application/json 那一发
[WebServer] {"msg":"probe833","原文长度":26,"头在":true}   ← 只换 content-type 那一发
```

我签的是那 26 个字符，带 JSON 类型那一发到服务端变成 32 字节——**发请求那一边把不是合法 JSON 的字符串按 JSON 又编了一次**（四个内层引号被转义，加外层一对正好 32）。签名算的是收到的字节，所以对不上。改的是探针，不是应用。

这条要连着记上一站撞过的一件事：**产物服务器的标准输出不进测试日志，错误输出进**。同一趟里 `webhook_received`（`console.log`）出现 0 次，那两条 `probe833`（`console.error`）出现 6 次。"日志里搜不到 X"永远不能当"X 没被写进日志"的证据；要钉这件事，就钉代码形状。

## 读数对不上时先看哪一处

| 症状 | 先看哪里 | 为什么 |
| --- | --- | --- |
| 同一条事件库里两行 | 占位那句有没有 `onConflictDoNothing` | 撞主键会变成 500，渠道一直重投 |
| 状态永远不动、全是迟到 | 时间有没有从秒换成毫秒 | 差一千倍，新事件也排在旧值后面 |
| 读不懂的事件第二条变成"重复事件" | 兜底主键是不是空串 | 空串是一条真主键，第二条会撞上它 |
| 白写一次看不出来 | 问 `pg_stat_user_tables` 的更新计数 | 写回同一个值，数据面完全没变化 |
| 探针发的字节和签的不一样 | 客户端会不会重编 JSON | 带 `application/json` 时非合法 JSON 的字符串会被再编一次 |

## 下一站

834 把三类事件的本地样本录下来，按"同一事件重放两次"与"较旧的事件后到"两种打法跑一遍，判据写成"响应码＋库里那一行"两条并列。

本轮跑法记录：新增 `map.ts`、`payload.ts`、`ingest.ts` 与 handler 接线；单测三个文件 15 条（映射 5、载荷 6、入账形状 4），合 313 条／49 个文件；端到端 `webhook-idempotency.spec.ts` 7 条，带库带密钥那趟 7 passed、不带那两个变量那趟 7 skipped；与 832、830 并跑 15 passed／1 skipped。`npm run check` 八项全 ok；全量端到端 255 passed／86 skipped，跳过分类是 要库 43／外部服务 19／内容目录 14／另一趟构建 10（新增那 7 条同时要库和密钥，脚本按第一条命中的规则记）。两次变异：去掉 `onConflictDoNothing` ⇒ ② 红，服务端回的是 `duplicate key value violates unique constraint "stripe_events_pkey"`；短路"没改就不写" ⇒ 只有白写那条红。都还原并复核过原样形状。上一站"200 之后库里 0 行"那条判据按新事实改了读法——它是当时的正确读数，现在过期了。

两处未闭合：`paused` 并到终态之后，站内"终态不再变"那条规则会把恢复事件挡在外面；"渠道状态与状态机结论不一致"只进日志、库里读不到，本站还没有能判它的地方。

## 本文依据

本站这一侧：

- 服务端文件的后缀与自动注册范围：https://nuxt.com/docs/4.x/guide/directory-structure/server
- 运行时配置与环境变量的对应关系（顶层键要带 `NUXT_` 前缀才被覆盖）：https://nuxt.com/docs/4.x/guide/configuration/runtime-config
- 库侧统计视图与 `pg_stat_force_next_flush`：https://www.postgresql.org/docs/current/monitoring-stats.html

第三方那一侧读的是 2026 年 10 月的在线文档，只用到字段定义与单位；订阅对象八个取值与"时间以秒计"两处的原文记录留在仓库 `.series/R107-830.md` 第六节：

- 事件对象（`id`／`type`／`created`，时间为秒）：https://docs.stripe.com/api/events/object
- 订阅对象（`status` 八个取值、`cancel_at_period_end`、`current_period_end`）：https://docs.stripe.com/api/subscriptions/object
- 发票对象（`status` 四个取值，与订阅那套是两套词表）：https://docs.stripe.com/api/invoices/object
- 重试与重新签名、验证签名的步骤：https://docs.stripe.com/webhooks
