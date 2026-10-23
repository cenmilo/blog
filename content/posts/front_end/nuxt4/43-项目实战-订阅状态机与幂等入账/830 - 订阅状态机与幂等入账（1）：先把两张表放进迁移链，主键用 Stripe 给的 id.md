---
url: /posts/0809bca5f6da430369884c9bb4014cec/
title: 830 - 订阅状态机与幂等入账（1）：先把两张表放进迁移链，主键用 Stripe 给的 id
date: 2026-11-04T09:00:00+08:00
updated: 2026-11-04T09:00:00+08:00
author: cmdragon

summary:
  摘要：仓库里已经有六张表，主键都是我们自己造的——uuid 或自增整数。这一站要加的第七张偏偏不能：那一串 id 是支付渠道给的，幂等就靠它天然唯一。这一篇把两张表放进迁移链的完整过程留下读数，包括文件名为什么长得像乱码、生成的 SQL 怎么和库里的记录对平、以及删一位用户时两张表为什么走向相反。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 数据库迁移
  - 幂等
  - 订阅付费
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## `db/tables/` 里已经有六张表，第七张的主键要故意不一样

打开 `db/tables/`，六个文件：`users.ts`、`sessions.ts`、`login-tokens.ts`、`audit-logs.ts`、`meter-events.ts`、`custom-types.ts`。它们的主键只有两种：`uuid('id')`（用户、券）和 `bigserial('id')`（审计、计量）；`sessions` 是个例外，它的 `id` 是 `text`——因为那串随机数是服务端自己发的。

这一站加两张表，主键都得是 `text`，而且不是我们自己造：

| 表 | 主键存的东西 | 为什么不能用 uuid |
| --- | --- | --- |
| `subscriptions` | Stripe 的订阅 id | 订阅的真名在渠道那边；我们再造一个 id 就等于每个事件都要先反查一次才知道改哪一行 |
| `stripe_events` | Stripe 的事件 id | 幂等靠的就是"同一个 id 来第二次插不进去"，这一位换成自增主键，重复事件就成了两行 |

## 两张表的定义怎么写

```ts
// db/tables/subscriptions.ts
export const subscriptions = pgTable('subscriptions', {
  id: text('id').primaryKey(),
  userId: uuid('user_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  customerId: text('customer_id').notNull(),
  status: text('status').notNull(),
  priceId: text('price_id').notNull(),
  currentPeriodEnd: timestamp('current_period_end', { withTimezone: true }),
  cancelAtPeriodEnd: boolean('cancel_at_period_end').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})

// db/tables/stripe-events.ts
export const stripeEvents = pgTable('stripe_events', {
  id: text('id').primaryKey(),
  type: text('type').notNull(),
  receivedAt: timestamp('received_at', { withTimezone: true }).notNull().defaultNow(),
  processedAt: timestamp('processed_at', { withTimezone: true }),
})
```

三处形状是有意选的，不是随手：

- **`status` 用 text 而不是 enum**。计量表那条理由在这里同样成立：状态值会加，enum 每加一个是改表结构。这一位还多一条，见后面"状态名不是渠道那套"。
- **两张表的删除方向相反**。订阅是"这个人自己的东西"，账号删了行就该跟着没，所以外键 `cascade`；事件是"渠道发过什么"这段事实，不属于任何一个用户，所以**一个外键都不挂**。这两条与前面那条线上量过的两种 `ON DELETE` 是同一族分工，但新表归哪一种要自己读一遍，不能引用旧结论。
- **`processed_at` 可空**。收到与处理完是两个时刻，中间失败了要看得出来；写成 `NOT NULL` 就没地方记"收到但没处理完"。

改完还得往 `db/schema.ts` 里加两行 `export *`——它是生成迁移时读的那一份出口，**漏一行等于这张表不存在**，而且报不出错。

## 生成、应用、以及对平

```bash
npm run db:generate     # 读 db/schema.ts 生成 SQL
npm run db:migrate      # 把没跑过的那几份应用到当前库
npm run check:migrations # 只读本地文件：.sql 与 meta/ 是否成对、顺序是否合法
```

`db:generate` 打印的表清单（本轮原文）：

```text
8 tables
ping 3 columns 0 indexes 0 fks
users 7 columns 0 indexes 0 fks
sessions 5 columns 0 indexes 1 fks
login-tokens 6 columns 0 indexes 0 fks
audit_logs 8 columns 0 indexes 1 fks
meter_events 4 columns 0 indexes 1 fks
subscriptions 9 columns 0 indexes 1 fks
stripe_events 4 columns 0 indexes 0 fks
[✓] Your SQL migration file ➜ db\migrations\0004_slimy_oracle.sql
```

**生成的文件名是随机词，里面不含表名**——`0004_slimy_oracle.sql`。这一点马上咬了我一次：判据里写了 `expect(文件名).toContain('subscriptions')`，于是永远红。要查表名只能查正文。

应用之后，库侧与文件侧要能对上号。`check:migrations` 给的是文件侧那半：

```text
0004_slimy_oracle      when=1792433299548  stmts=3  tables=8
                       sha256=4c8bf8bcaa2def0a6a6a0448916b51bd47bba0b4a6f822094bb0b8470d95e386
```

它最后一行会提醒：库侧那半段要自己在库上查。查出来是这样（本轮读数）：

```json
[文件与库对账] {"journal里条数":5,"库侧条数":5,
  "文件sha256":"4c8bf8bcaa2def0a6a6a0448916b51bd47bba0b4a6f822094bb0b8470d95e386",
  "库侧hash":"4c8bf8bcaa2def0a6a6a0448916b51bd47bba0b4a6f822094bb0b8470d95e386","hash相等":true,
  "journal_when":1792433299548,"库侧_when":"1792433299548","when相等":true}
```

两张表都在库里了，列形状也是想要的那样：

```json
[建表后·列形状] {"stripe_events":["id:text:NO","type:text:NO","received_at:timestamp with time zone:NO:now()",
  "processed_at:timestamp with time zone:YES:(无默认)"],
  "外键":{"conname":"subscriptions_user_id_users_id_fk","confdeltype":"c"}}
```

`confdeltype` 那位是 PostgreSQL 目录表里的删除动作码，`c` 就是 ON DELETE CASCADE。

## 引入前的账：这个地址还不存在

上一站（829）收口时会员区那一串接口里没有支付那一路，所以这一站先把它钉住：

```json
[建表前·地址] [{"方法":"get","状态":404,"正文字节":258,"allow头":"(没有这一位)","请求号在":true},
  {"方法":"post","状态":404,...},{"方法":"put","状态":404,...}]
```

3 种方法都是 404，响应里**没有 `allow` 那一位**。这值得记一句：很多人以为"路径存在但方法不对"才 404／405，而 Nitro 对没注册的路径任何方法都回 404，所以拿 404 探测"这个接口有没有建"在本机是可靠的，拿 HEAD 探测则不可靠（同一件事在会员区那个接口上量过）。库里那半边的引入前读数：

```json
[{"command":"SELECT","rowCount":0,"rows":[]}]   // subscriptions、stripe_events 都还没出现
[{"现有表数":6}]                                 // 应用迁移之前
```

## 删一位用户，两张表走向相反

这是新表上最该读到的一条，因为它把"幂等表不该被牵连"这件事变成了数：

```json
[新表·两种分工] {"删前":{"订阅":1,"事件":1},"删后":{"订阅":0,"事件":1}}
```

0 与 1 同时成立才说明两半都在真读：如果两张表都挂 cascade，这里会读到 `{0,0}`；如果订阅那条外键没生效，会读到 `{1,1}`。这一对数是判据里最不怕空转的形状。

## 状态名不是渠道那套

顺手把一件容易混的事量清楚了。本站 `status` 存的是自己的五个名字：`none`／`active`／`past_due`／`cancellation_scheduled`／`inactive`。而 Stripe 的订阅对象那一页对 `status` 给的是另一套：

> Possible values are `incomplete`, `incomplete_expired`, `trialing`, `active`, `past_due`, `canceled`, `unpaid`, or `paused`.

**两边只有两个词重合**。所以这张表里的 `status` 一旦写成 enum 并照抄渠道的取值集合，下一次渠道加一个值（或者我们做一次状态归并，比如把 `canceled` 和过期都收敛成 `inactive`）就是改表结构。用 text、把映射放在应用侧的纯函数里（下一站就做），改起来只动代码。

## 两次变异，以及一次没做的

- 把 `stripe_events` 的主键从 `text('id')` 改成 `uuid('id')`：单测立刻红 `stripe_events 的主键不是 text 主键: expected false to be true`，打印那一行 `id: uuid('id').primaryKey()`。改回后五条全过。
- 在 `0004_slimy_oracle.sql` 末尾加一行注释（库里没动）：对账那一档红 `Expected: "a4bc44fa…" Received: "4c8bf8bc…"`，而其它三档照旧绿 ⇒ 说明这一档真的在比对文件与库，而不是顺手读两个数。删掉那行注释后 hash 回到 `4c8bf8bc…`，四档全绿。
- **没做的那次**：让 `processed_at` 真的变成 `NOT NULL`、或者给 `stripe_events` 挂上外键，都要在这台开发库上做破坏性 DDL。本轮没做，所以那两档（列形状、两种分工）的非空断言靠的是"读到了具体值"（`YES`、`false`、`c`、以及上面那对 `0,1`）而不是变异撞红。这一条记在未验里，谁要在自己的库上补，改完记得把库退回迁移状态。

## 读数对不上时先看哪一处

| 症状 | 先看哪里 | 为什么 |
| --- | --- | --- |
| 生成的 SQL 里没有新表 | `db/schema.ts` 有没有那两行 `export *` | 生成器只读那份出口，漏一行不报错 |
| 迁移文件名找不到表名 | 查正文，别查文件名 | 文件名是随机词 |
| 库侧 hash 与文件不等 | 那份迁移在这个库上跑过吗 | 不等就是 drift：要么漏跑，要么有人手改过库 |
| `confdeltype` 不是 `c` | 表定义里 `onDelete` 写没写 | 默认是没动作（`a`），不是 cascade |
| 删用户后两张表都空 | `stripe_events` 被挂了外键 | 渠道历史不该跟着账号走 |
| 地址读到 405 | 你打的路由其实存在 | Nitro 对没注册的路径一律 404 |

## 下一站

状态与时间戳的迁移规则先写成不碰库的纯函数（讲聚合函数那一站就是这么立的），期望值手写；"旧事件不许把新状态改回旧值"这句要在函数上先证一次，再谈接口。

本轮跑法记录：新单测 **5 passed**、新端到端档默认趟 **1 passed／3 skipped**、带库趟 **4 passed (39.3s)**；两次变异各撞一条红，还原后复跑再次全绿；`npm run check` 八项全 ok（单测从 281 条增至 **286 条**、`check:migrations` 问题 0）；整站端到端 **254 passed／75 skipped**（跳过分类：要库 43、要内容目录读得到 14、要另一趟构建的开关 10、要外部服务变量 8）。探针行用完即删，两张新表收尾时都是 0 行。

## 本文依据

本站这一侧：

- 运行时配置与环境变量名的对应：https://nuxt.com/docs/4.x/guide/going-further/runtime-config
- 服务端路由与自动导入（`server/api` 的注册方式）：https://nuxt.com/docs/4.x/guide/directory-structure/server

第三方那一段（本项目里第三方断言单独列，并标注取证方式）：

- Stripe 订阅对象 `status` 的取值：https://docs.stripe.com/api/subscriptions/object（原话见上文那一节）
- Stripe 事件对象 `id`／`type`／`created`：https://docs.stripe.com/api/events/object（该页文档的 API 版本写作 2019-02-19；页内没有关于重放的承诺）
- PostgreSQL 目录表 `pg_constraint` 的 `confdeltype`：https://www.postgresql.org/docs/current/catalog-pg-constraint.html
- Drizzle 的删除动作类型（文档站那一页取不到正文，改引本机包内定义）：`node_modules/drizzle-orm/pg-core/foreign-keys.d.ts:4` 写着 `export type UpdateDeleteAction = 'cascade' | 'restrict' | 'no action' | 'set null' | 'set default';`，第 19 行是 `onDelete?: UpdateDeleteAction;`，包版本 drizzle-orm 0.45.4
