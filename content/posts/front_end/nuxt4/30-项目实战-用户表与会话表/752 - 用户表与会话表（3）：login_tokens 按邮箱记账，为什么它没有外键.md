---
url: /posts/728d8115969a9ff035b153d953b8a4a2/
title: 752 - 用户表与会话表（3）：login_tokens 按邮箱记账，为什么它没有外键
date: 2026-10-22T11:00:00+08:00
updated: 2026-10-22T11:00:00+08:00
author: cmdragon

summary:
  摘要：生成器把 schema 念一遍时，`login_tokens` 那一行写的是六列、零索引、零外键——这个仓库里唯一一张不挂外键的真表。这一站先量清它凭什么能被使用：券的号先于人的号存在；再顺手把那条外键加上，看生成出什么句子、类型那边又是否有任何变化。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 登录链接
  - 数据表设计
  - Drizzle
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 六列、零索引、零外键

`npm run db:generate` 在没有改动时会把 schema 念一遍，这张表那行是：

```text
login_tokens 6 columns 0 indexes 0 fks
```

前后几张都是 `1 fks`（`sessions` 挂 `user_id`、`audit_logs` 挂 `actor`、`meter_events` 挂 `user_id`），只有它是 `0`。这不是漏了——这是这个仓库里唯一一处**故意不挂外键**的关系，而它最容易被"顺手加一个更严谨"改错。

先把它凭什么能用看清，再真的加一次，看会付出什么。

## 六列先认全

| 这一列 | 生成的 SQL（`0001_core_tables.sql` 第 29 到 37 行） | 谁负责给值 |
| --- | --- | --- |
| `id: uuid('id').primaryKey().defaultRandom()` | `"id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL` | 库（和 `users.id` 同一位规矩） |
| `email: citext('email').notNull()` | `"email" "citext" NOT NULL` | 应用 |
| `tokenHash: text('token_hash').notNull().unique()` | `"token_hash" text NOT NULL` ＋ `CONSTRAINT "login_tokens_token_hash_unique"` | 应用，而且**存的是摘要** |
| `expiresAt: timestamp(...).notNull()` | `"expires_at" timestamp with time zone NOT NULL` | 应用算好写进来 |
| `usedAt: timestamp(...)` | `"used_at" timestamp with time zone` | 可空——空的意思是"还没被用过" |
| `createdIp: inet('created_ip').notNull()` | `"created_ip" "inet" NOT NULL` | 应用 |

关键在第一行和第二行的关系：这张表**记的是收件地址，不是用户号**。库里存的是明文 token 的 sha256 摘要（明文只在发信那一刻存在，见 `server/services/auth/token.ts`），拿到整张表也换不出任何一条还能用的链接。

## 一次性靠一条 UPDATE，不靠"先查再改"

兑换那一步在 `server/services/auth/verify.ts` 第 32 到 44 行：

```ts
export async function claimLoginToken (token: string, now: Date = new Date ()): Promise<ClaimedToken | null> {
  const [row] = await useDb()
    .update(loginTokens)
    .set({ usedAt: now })
    .where(and(
      eq(loginTokens.tokenHash, hashToken(token)),
      isNull(loginTokens.usedAt),
      gt(loginTokens.expiresAt, now),
    ))
    .returning({ email: loginTokens.email })

  return row ? { email: row.email } : null
}
```

三个条件全写进同一句 `UPDATE` 的 `where`，**改到行才算占到**。两个请求同时拿同一枚 token，数据库只会让其中一个改中——不需要应用层去抢锁，也没有"先查出来是未用、再去改"之间那个竞争窗口。没改中就走同一个 `null`，于是"不存在""过期了""用过了"三种情况对外是同一句文案、同一个 code。

再看这张表的另一条路径，第 50 到 54 行：

```ts
export async function revokeOtherTokens (email: string): Promise<void> {
  await useDb()
    .delete(loginTokens)
    .where(and(eq(loginTokens.email, email), isNull(loginTokens.usedAt)))
}
```

成功登录时把同址其它未用的券删掉。**这两条路径里没有任何一处按用户号查券**——外键该挂的那个键，代码从来不用。

## 类型那一趟：四列必填、两列可省

临时探针五段声明（跑完就删），`npx nuxt typecheck` 的 `rc=2` 输出只有两行：

```text
db/probe-token-types.ts(23,14): error TS2741: Property 'tokenHash' is missing in type '{ email: string; expiresAt: Date; createdIp: string; }' but required in type '{ email: string; tokenHash: string; expiresAt: Date; createdIp: string; id?: string | undefined; usedAt?: Date | null | undefined; }'.
db/probe-token-types.ts(30,14): error TS2741: Property 'createdIp' is missing in type '{ email: string; tokenHash: string; expiresAt: Date; }' but required in type '{ email: string; tokenHash: string; expiresAt: Date; createdIp: string; id?: string | undefined; usedAt?: Date | null | undefined; }'.
```

`required` 那串把形状整个列出：必填四位（`email`、`tokenHash`、`expiresAt`、`createdIp`），可省两位（`id` 因为库有默认值、`usedAt` 因为可空）。对一下签发那段代码，`server/services/auth/token.ts` 第 59 到 62 行的 `.values()` 正好这四个键——又一次，判据是实现本来就这么写的。

还有一段**故意不报错**的负对照：`createdIp: '这不是一个 IP'` 顺利通过。`inet` 列在类型里也是 `string`（和 [上一站](/posts/8702668a938b19689e83ea3783a7ece5/)那条 `uuid` 的结论同形）。要挡非法地址，得自己在应用层判；库里那道只在真落库时生效。

## 顺手把外键加上，会生成什么

这一站也做一次"反事实"的实验，全程只写文件、不碰库：给 `email` 那一列加一句指向 `users.email` 的外键，然后跑生成。

表统计那行立刻变成 `login_tokens 6 columns 0 indexes 1 fks`，新文件 `0004_shocking_toxin.sql` 是**177 字节、零个分句标记、一段**，全文一句：

```sql
ALTER TABLE "login_tokens" ADD CONSTRAINT "login_tokens_email_users_email_fk" FOREIGN KEY ("email") REFERENCES "public"."users"("email") ON DELETE no action ON UPDATE no action;
```

四条读数：

- **加一条外键是一条语句**，不像上一站"改一条外键"要两条（先删再建）。
- 约束名是拼出来的：`表名_列名_引用表_引用列_fk`。所以列名和表名一旦改，约束名跟着改——别把它当稳定标识符写进迁移脚本手工执行。
- 引用目标必须是唯一的列。这里能成立，正是因为 `users.email` 上有那条 `UNIQUE("email")`——[750 那一站](/posts/94d2feb78b4176ba453ad7f6fc86f9a9/)列出的唯一约束，在这里成了外键的前置条件。
- **类型那一趟一行都没变**：加外键前后各跑一次 `nuxt typecheck`，两次的报错行逐字相同。和 `onDelete` 一样，外键这件事在 TypeScript 里完全隐形。

那为什么不加？两条理由都能从代码顺序里查出来，不是风格偏好。

**第一条：券的号先于人的号存在。**发信那一步要落一行券，而账号可能是**兑换那一刻才建的**——`server/api/auth/verify.get.ts` 第 61 行是 `upsertUserByEmail(claimed.email)`，"没这个邮箱就建一个"。如果 `login_tokens.email` 上挂着外键，第一次用某个新邮箱申请链接的人会先在发信那一步就被库挡住：那行券引用了一个还不存在的地址。这条链路最坏的形状不是报错难看，而是新用户根本收不到登录邮件。

**第二条：删账号这件事不该被券挡住，也不该顺手删掉券的历史。**那句生成出来的规则是 `ON DELETE no action`——按字面读，它意味着删用户时数据库不会替你处理那些券（真删一次会怎样、报哪个码，本轮没连库，所以这句只作"待你验"的推断）。换成 `cascade` 则是另一种坏：券行跟着人没，而那张表是排查"谁给这个邮箱发过链接"的凭据。两种都不想要，于是这张表故意游离，靠 `email` 这个业务键连回去。

实验照例撤干净并核对：`db/tables/login-tokens.ts` 与备份 `cmp` 逐字节相同（`e5752e62ab0d`）、新 `.sql` 与 `meta/0004_snapshot.json` 删净、journal 回到 `91a102b887d6`／四条、`node scripts/check-migrations.mjs` 输出 `问题 0`，再跑一次生成回到 `No schema changes, nothing to migrate`。

## citext 之外还有一层归一

`email` 用 `citext`，是为了让大小写变体在比较与唯一约束上算同一个地址。但库里那一层管不到空白：`shared/utils/normalize-email.ts` 只做两件事——去首尾空白、把 `@` 后面降小写（本地部分保持原样）。这几条都钉在 `test/unit/auth-schema.test.ts` 的用例名里：「域名降小写，本地部分保持原样」「归一是幂等的：连着做两次结果不变」「自己不 trim：带空白的合法地址会被判非法」「先归一再校验：大小写混排带空白的输入能过，出来的地址已归一」。

顺序是"先归一，再进 schema，再入库"，三层各管一段：空白、大小写、以及 citext 的比较语义。少任何一层，`A@B.co` 与 `a@b.co` 就会变成两个账号。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 新邮箱点"发登录链接"就 500，而老邮箱正常 | 给 `login_tokens.email` 加了外键，而账号是兑换时才建的 | 别加。这张表按业务键连，不靠引用完整性 |
| 同一枚链接点第二次也登录成功 | 一次性是靠那句带条件的 `UPDATE` 实现的，被拆成"先查再改"就丢了原子性 | 保持一条语句里带 `used_at IS NULL` 与有效期条件 |
| 邮件里那条链接里的 token 在库里搜不到 | 库里存的是 sha256 摘要，不是明文 | 用同一份 `hashToken()` 换算之后再比 |
| 传了非法的 `created_ip` 值，类型检查也绿 | `inet` 列在类型里就是 `string` | 应用层先校验地址形状；库里那道只在落库时生效 |
| 同一个地址的大小写变体注册成两个账号 | `email` 不是 `citext`，或者入库前没归一 | 两件事都要：列类型 ＋ `normalizeEmail` |

## 自己在自己的库上验

本轮没有连库，下面三条都没有读数，是留给你的动作（前两条只读，第三条要删数据，务必用可丢弃的库）：

1. 确认它真的没有外键：`select conname, contype from pg_constraint where conrelid = 'login_tokens'::regclass;`——预期只有那条 `token_hash` 的 unique 与主键，没有 `FOREIGN KEY`。
2. 确认摘要那一列的形状：`select length(token_hash) from login_tokens limit 1;` 应当是 64。
3. 拿一个有未使用券的邮箱删掉它的账号，看会不会被挡；再拿那枚券兑换一次，看是不是**建出一个新号**（`upsertUserByEmail` 的分支决定这件事，而它的 id 和旧账号不同）。这一条是本站推断里唯一没量的那条，也是 812 那条线要正面处理的行为。

## 这一站的边界

一次库连接都没有建立（连库与写库要先同意）。所以"删账号会不会被券挡住""券能不能重建账号"这两件事在正文里都是**代码顺序上的推断＋待你验**，不是读数。类型与生成两头的读数是本地跑出来的，全在上文，还原逐字节核对过。

下一步 [753《用户表与会话表（4）：后面两份迁移各加了什么：两种 ON DELETE 的分工》](/posts/85041fb63edb1bf9267bd29d431935a2/)：`0002` 与 `0003` 两份文件形状一模一样，只差那两个词——这一站把 `set null` 落到类型上看它是什么形状，再试一个矛盾组合，看有没有人拦。

## 本文依据

- 这张表的列与那条唯一约束：本仓库 `db/tables/login-tokens.ts`、`db/migrations/0001_core_tables.sql` 第 29-37 行
- 一次性与同址撤券的实现、发信只写摘要：`server/services/auth/verify.ts:32-54`、`server/services/auth/token.ts:20-73`
- 账号在兑换那一步才建：`server/api/auth/verify.get.ts:61` 调 `upsertUserByEmail`，其定义在 `server/services/auth/verify.ts:57`
- 邮箱归一的三条行为与 zod 口径：`shared/utils/normalize-email.ts`、`test/unit/auth-schema.test.ts` 的用例名
- 外键与删除规则不进类型、`uuid`／`inet` 列在类型里是 `string`：[751《用户表与会话表（2）：sessions 的外键为什么是 cascade》](/posts/8702668a938b19689e83ea3783a7ece5/)
- 唯一约束是外键的前置条件、`citext` 的类型来源：[750《用户表与会话表（1）：一张 users 表要哪七列，默认值钉在哪》](/posts/94d2feb78b4176ba453ad7f6fc86f9a9/)
- 本轮读数：`npm run db:generate` 的两行表统计（`0 fks` 与 `1 fks`）、`0004_shocking_toxin.sql` 的 177 字节与单句形状、两趟 `npx nuxt typecheck` 的报错对比、还原后的 `问题 0` 与 `No schema changes`
