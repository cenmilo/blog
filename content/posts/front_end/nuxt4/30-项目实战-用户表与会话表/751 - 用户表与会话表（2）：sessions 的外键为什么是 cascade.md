---
url: /posts/8702668a938b19689e83ea3783a7ece5/
title: 751 - 用户表与会话表（2）：sessions 的外键为什么是 cascade
date: 2026-10-22T10:00:00+08:00
updated: 2026-10-22T10:00:00+08:00
author: cmdragon

summary:
  摘要：`db/tables/sessions.ts` 只有十五行、五列，其中一列带着外键和它的删除规则。这一站改的只有那一个词，然后分两头看：类型那一趟的报错逐字没变，SQL 这一头却生成出两条语句。看完就知道这条规矩归谁管、什么情况下它替不了应用自己写的那两条删除。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 外键
  - 会话表
  - Drizzle
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 一个词，两头看

[上一站](/posts/94d2feb78b4176ba453ad7f6fc86f9a9/)量的是列的默认值，这一站量的是**外键的删除规则**。`db/tables/sessions.ts` 全文十五行，值得琢磨的是第九到十一行：

```ts [db/tables/sessions.ts 里那个外键]
  userId: uuid('user_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
```

"删用户的时候他的会话跟着没"这句话，到底钉在哪一层？方法很笨但有效：**把 `cascade` 改成别的词，然后分别跑类型检查和生成**，看哪一边会动。

## 五列先认全

| 这一列 | 生成的 SQL（`0001_core_tables.sql`） | 谁负责发值 |
| --- | --- | --- |
| `id: text('id').primaryKey()` | `"id" text PRIMARY KEY NOT NULL` | 应用。号在 `server/utils/http/session-cookie.ts` 第 37 行由 `randomUUID()` 发出，浏览器 cookie 里存的就是它 |
| `userId: uuid('user_id').notNull().references(...)` | `"user_id" uuid NOT NULL` ＋ 文件末尾那句 `ALTER TABLE "sessions" ADD CONSTRAINT …` | 必须是 `users` 里存在的号，否则库里就拒 |
| `createdAt: timestamp(...).notNull().defaultNow()` | `"created_at" timestamp with time zone DEFAULT now() NOT NULL` | 库 |
| `expiresAt: timestamp(...).notNull()` | `"expires_at" timestamp with time zone NOT NULL` | 应用（必填，没有默认值） |
| `lastSeenAt: timestamp(...).notNull().defaultNow()` | `"last_seen_at" timestamp with time zone DEFAULT now() NOT NULL` | 库 |

`npm run db:generate` 在没有改动时会把 schema 念一遍，这一张那行是：

```text
sessions 5 columns 0 indexes 1 fks
```

`1 fks` 那一位就是外键，而 `0 indexes` 说明它没有额外索引——只有主键和那条外键约束。

顺带注意第一列：库里的类型是 `text`，不是 `uuid`。因为这个号是应用发的、只需要唯一，不需要库生成，也不需要 uuid 那种类型检查。

## 类型那一趟：改那个词，一行都没变

先造一个只看类型的临时文件（跑完就删，五段声明），跑 `npx nuxt typecheck`：

```text
db/probe-fk-types.ts(22,14): error TS2741: Property 'id' is missing in type '{ userId: string; expiresAt: Date; }' but required in type '{ id: string; userId: string; expiresAt: Date; createdAt?: Date | undefined; lastSeenAt?: Date | undefined; }'.
db/probe-fk-types.ts(28,14): error TS2741: Property 'expiresAt' is missing in type '{ id: string; userId: string; }' but required in type '{ id: string; userId: string; expiresAt: Date; createdAt?: Date | undefined; lastSeenAt?: Date | undefined; }'.
```

`rc=2`，输出只有这两行。三件事当场清楚：

- **必填的是 `id`、`userId`、`expiresAt` 三位**，`createdAt` 与 `lastSeenAt` 因为带默认值可以省——和上一站那条判据完全一致。
- 探针里故意写了 `userId: '这不是一个 uuid'`，**它没报错**。`uuid` 列在类型里就是 `string`，一个不合法的号能一路走到库里才被拒。
- 现在把 `onDelete: 'cascade'` 改成 `'no action'`，同一趟再跑一次：报错行与上面两行逐字相同（`diff` 出来是空的）。

最后这条是本站最要紧的读数：**外键的删除规则根本不进类型**。它对 `sessions.$inferInsert`、`$inferSelect` 没有任何影响——你在 TypeScript 里看不出这个用户的会话会跟着他没。

## SQL 这一头：一次改动作两条语句

同一个改动拿去生成，得到 `0004_messy_blade.sql`——**263 字节、一个 `--> statement-breakpoint`、两段**，全文是：

```sql
ALTER TABLE "sessions" DROP CONSTRAINT "sessions_user_id_users_id_fk";
--> statement-breakpoint
ALTER TABLE "sessions" ADD CONSTRAINT "sessions_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE no action ON UPDATE no action;
```

三点值得记住：

1. **改外键不是"改"，是先删约束再建约束**。约束名复用同一个 `sessions_user_id_users_id_fk`，位置换了一条新的。
2. 两段执行——这又一次对上前面那条规律：**分句标记数 = 语句数 − 1**。
3. 中间那个"外键暂时不存在"的时刻，从已证事实能推出结论：上一站读到包里是把一份迁移的**全部语句放在同一个事务里**执行的，所以别的会话看不到这个空档。这是推论，本轮没有造现场去观察它。

实验之后全部还原，逐条核对过：`sessions.ts` 与备份 `cmp` 逐字节相同（sha256 前 12 位 `1e9a546a7afd`）、`meta/_journal.json` 回到 `91a102b887d6`／四条记录、四份 `.sql` 的哈希仍是 `6153a23bc13b`／`769be43ce241`／`30ceae56796b`／`cf7ba90d4e2b`，上一站那条自检脚本输出 `问题 0`，再跑一次生成回到 `No schema changes, nothing to migrate`。

## 那 cascade 到底替谁省事

它管的是**父行被删**那一路：`users` 里那一行没了，`sessions` 里属于他的行跟着没，不需要谁先去清。

但应用里有两种删除是它替不了的，仓库里就是两条真代码：

```ts [server/services/auth/session.ts 的两条删除]
export async function deleteSessionBySid (sid: string): Promise<void> {
  await useDb().delete(sessions).where(eq(sessions.id, sid))
}

export async function deleteSessionsByUserId (userId: string): Promise<number> {
  const rows = await useDb()
    .delete(sessions)
    .where(eq(sessions.userId, userId))
    .returning({ id: sessions.id })

  return rows.length
}
```

第一个是登出（只踢这一台设备），第二个是全端下线（踢掉这个人的所有设备）。**这两条删的时候账号还在**——`cascade` 只在账号真被删那一刻起作用，所以应用自己那两条不能省。反过来，删账号的那条路径不需要写"先删会话"，那是 `cascade` 的职责。

还有一处对照很能说明问题。登录时的轮换（`server/services/auth/verify.ts` 第 91 到 94 行）：

```ts
await useDb().transaction(async (tx) => {
  if (staleSid) { await tx.delete(sessions).where(eq(sessions.id, staleSid)) }
  await tx.insert(sessions).values({ id: newSid, userId, expiresAt })
})
```

`values()` 里正好三个键：`id`、`userId`、`expiresAt`——和类型探针判出来的三列必填逐字相同。这条判据不是先设计再实现的，是实现本来就这么写的，类型只是把它说出来了。

同一条线上另一种选择已经存在：`audit_logs.actor` 是可空的 FK 加 `set null`——删用户时痕迹要留着。为什么两处选得不一样，第 4 步讲。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 插入会话报 `user_id` 不能为 null 或者违反外键 | 没给这个键，或者给的号在 `users` 里不存在 | 先建用户拿到 `id` 再建会话；外键和 `NOT NULL` 是两道不同的闸 |
| 删了用户，他的会话没了，但登录券还在 | `login_tokens` 上没有外键，`cascade` 管不到它 | 第 3 步专门讲那张表为什么故意没有外键 |
| 改了 `onDelete` 之后库里行为没变 | 生成只写文件，执行要 `npm run db:migrate` | 跑之前先确认那个库是可以扔的 |
| 写了个不像 uuid 的会话号，typecheck 也绿 | `uuid` 列在类型里就是 `string`（本轮那条探针没报错） | 要在应用层挡，就自己加形状校验；库里那道只在真正落库时生效 |
| 审计行跟着用户一起没了 | 那条 FK 写成了 `cascade` | 追责用的表要 `set null` 加可空列，别照抄会话表 |

## 自己在自己的库上验

本轮没有连库（下面第 1、2 条只读，第 3、4 条是删数据，务必在可丢弃的库上跑）：

1. 这条外键在库里是什么规则：`select conname, confdeltype from pg_constraint where conrelid = 'sessions'::regclass and contype = 'f';`。`confdeltype` 会给出一个字符，三种规则是三个不同字符——**具体对应关系本轮读不到（没连库），这里不做断言**，你跑出来就知道。
2. 那张表有没有索引：`select indexname from pg_indexes where tablename = 'sessions';`，预期只有主键那一条。
3. `cascade` 的体感：插一个用户加两条会话，然后 `delete from users where id = ...`，再 `select count(*) from sessions where user_id = ...`——归零，而你全程没写过删会话的语句。
4. 把 `ON DELETE` 换成 `no action` 之后同样的动作应当被外键挡住（报违反外键）。这一条正好是本站那次生成的反面。

## 这一站的边界

一轮库连接都没建立：开头试着读一次列信息被安全层挡下（连库要先同意），**没有换写法绕**。所以上面第 1 到 4 条的读数都是预测，不是量到的；类型与 SQL 两头的读数是本地量出来的，全在上文。真删用户看谁跟着没，属第 4 步；整条链在空库上跑到哪句停，属第 5 步。

下一步 [752《用户表与会话表（3）：login_tokens 按邮箱记账，为什么它没有外键》](/posts/728d8115969a9ff035b153d953b8a4a2/)：生成器念这张表时那一行是「6 columns 0 indexes 0 fks」——整个仓库唯一一张不挂外键的真表。这一站真的把它加上，看付出什么代价。

## 本文依据

- 列的定义与那条外键的句子：本仓库 `db/tables/sessions.ts`、`db/migrations/0001_core_tables.sql`（表定义在第 21-27 行，外键在第 39 行）
- 会话号由谁发、两条删除分别删什么、轮换的事务形状：`server/utils/http/session-cookie.ts:37`、`server/services/auth/session.ts:44-63`、`server/services/auth/verify.ts:91-94`
- 分句标记切段、一份迁移在同一个事务里执行、生成只写文件不碰库：[748《开发库与第一份迁移（5）：落库与记账：跑过的为什么不会再跑一遍》](/posts/a8f208c67579670627989f47b7758406/) 与 [749《开发库与第一份迁移（6）：这套文件怎么守住：入库、对账与类型护栏》](/posts/02c4422156fea45f01ecd620da8d2e9e/)
- 类型检查要单独跑、`nuxt dev` 与 `nuxt build` 默认不查类型：<https://nuxt.com/docs/4.x/guide/concepts/typescript>
- 本轮读数：两趟 `npx nuxt typecheck` 的报错对比、`0004_messy_blade.sql` 的 263 字节与两段形状、`npm run db:generate` 的表统计与还原后的 `No schema changes`、`node scripts/check-migrations.mjs` 的 `问题 0`
