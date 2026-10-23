---
url: /posts/94d2feb78b4176ba453ad7f6fc86f9a9/
title: 750 - 用户表与会话表（1）：一张 users 表要哪七列，默认值钉在哪
date: 2026-10-22T09:00:00+08:00
updated: 2026-10-22T09:00:00+08:00
author: cmdragon

summary:
  摘要：`db/tables/users.ts` 里七行字段，`0001_core_tables.sql` 里那段 `CREATE TABLE "users"`。这一站逐行对它们，并把一个容易含糊的问题量清楚：列的默认值到底钉在库里还是代码里。做法是两次可撤销的生成——去掉一个默认值、再换一个默认值，看生成器写出什么句子，再看它改不改旧文件。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 数据表设计
  - Drizzle
  - PostgreSQL
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 这条线从一张真表开始

打开 `db/tables/users.ts`，`pgTable` 里是七行字段；打开 `db/migrations/0001_core_tables.sql`，第 9 到 19 行是它对应的 `CREATE TABLE "users"`。[上一站](/posts/02c4422156fea45f01ecd620da8d2e9e/)把"文件成对"守住了，这一站开始问内容：一张用户表要哪几列、每列的规矩钉在**哪一层**。

钉在库里和钉在代码里不是同一件事。默认值钉在库里，插入时不写它也有值；钉在代码里，绕过应用直接写库的那一路就没人管。这个区别靠读代码读不出来，得改一次看看生成器写什么句子。

两个词先说清：`DDL` 是建表、改表那一类语句；`DEFAULT` 是列上的默认值，插入时不给这一列，数据库就填它。

## 七列逐行对

| `db/tables/users.ts` 那一行 | `0001_core_tables.sql` 里的那一句 | 为什么这么定 |
| --- | --- | --- |
| `uuid('id').primaryKey().defaultRandom()` | `"id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL` | 号由库生成，应用不自己拼 uuid。那个函数来自 `pgcrypto` 扩展，所以扩展行必须先生效 |
| `citext('email').notNull().unique()` | `"email" "citext" NOT NULL` ＋ `CONSTRAINT "users_email_unique" UNIQUE("email")` | `citext` 是大小写不敏感的文本：写进去原样保存，比较和唯一约束按不区分大小写算。邮箱要的正是这个 |
| `text('name')` | `"name" text` | 可空。从登录链接进来的人可以没名字 |
| `userRole('role').notNull().default('free')` | `"role" "user_role" DEFAULT 'free' NOT NULL` | 值域是枚举，默认档 `free` |
| `text('github_id').unique()` | `"github_id" text` ＋ `CONSTRAINT "users_github_id_unique" UNIQUE("github_id")` | 可空又唯一：没绑 GitHub 的那些行都留 `NULL`。至于多条 `NULL` 算不算重复，本项目没验，读者可在自己的库上一条 `insert` 就测出来 |
| `timestamp('created_at', { withTimezone: true }).notNull().defaultNow()` | `"created_at" timestamp with time zone DEFAULT now() NOT NULL` | 带时区，默认取库的 `now()` |
| `timestamp('banned_at', { withTimezone: true })` | `"banned_at" timestamp with time zone` | 封号记的是"什么时候封的"这个时间点，不是一个布尔开关——能留痕，也撤得掉 |

形状上还有一处差别值得注意：`users` 的两条唯一约束写在 `CREATE TABLE` 的括号**里面**，而 `sessions` 的外键是同一份文件末尾那句独立的 `ALTER TABLE "sessions" ADD CONSTRAINT …`。建表时就带上的约束和后来补的约束，生成位置不同。

## 枚举那三个值进了两边

`userRole` 这一行是 `pgEnum('user_role', ['free', 'member', 'admin'])`，生成出来的 SQL 是：

```sql
CREATE TYPE "public"."user_role" AS ENUM('free', 'member', 'admin');
```

上一站量过切段的规矩：分句标记把一份文件切成几次执行。这一份里，两条手写的 `create extension` 和这句 `CREATE TYPE` 在**同一段**，所以它们一定先于 `CREATE TABLE "users"` 生效——枚举名在建表时就得存在。

值域不只钉在库里，它也进了 TypeScript。造一个只碰类型的临时文件（跑完就删，全文如下）：

```ts [db/probe-insert-types.ts（本轮探针，已删）]
// 临时探针（跑完即删）：只看类型，不连库、不建表。
// 前两行应当通过（三列都有 DEFAULT，插入时可以不给）；后两行故意写错，报错内容就是本轮的读数。
import { users } from './tables/users'

type Row = typeof users.$inferSelect
type NewUser = typeof users.$inferInsert

export const okOmitDefaults: NewUser = {
  email: 'probe@example.com',
  name: null,
  githubId: null,
  bannedAt: null
}

export const okRole: Row['role'] = 'free'

export const missingEmail: NewUser = {
  name: null,
  githubId: null,
  bannedAt: null
}

export const badRole: Row['role'] = 'superuser'
```

`npx nuxt typecheck` 的完整输出只有两行，正好是故意写错的那两处：

```text
db/probe-insert-types.ts(17,14): error TS2741: Property 'email' is missing in type '{ name: null; githubId: null; bannedAt: null; }' but required in type '{ email: string; id?: string | undefined; name?: string | null | undefined; role?: "free" | "member" | "admin" | undefined; githubId?: string | null | undefined; createdAt?: Date | undefined; bannedAt?: Date | ... 1 more ... | undefined; }'.
db/probe-insert-types.ts(23,14): error TS2322: Type '"superuser"' is not assignable to type '"free" | "member" | "admin"'.
```

第 23 行那句说明：`pgEnum` 那三个值就是 `role` 的类型。改清单，全仓库对 `role` 赋值的地方当场红。库里那一侧同样会拒——值不在清单里的 `insert` 会被 Postgres 判错；这一趟没连库，那句原文留给读者（文末那条只读语句之外，再补一条 `insert` 就看得见）。

## 默认值钉在哪：两次生成

把 `db/tables/users.ts` 里的 `.default('free')` 去掉，跑一次 `npm run db:generate`：

```text
[✓] Your SQL migration file ➜ db\migrations\0004_nebulous_korath.sql
```

那份新文件全文只有一句，**53 字节**：

```sql
ALTER TABLE "users" ALTER COLUMN "role" DROP DEFAULT;
```

再把 `.default('free')` 换成 `.default('member')`，又生成一份 `0005_jazzy_killraven.sql`，全文一句、**61 字节**：

```sql
ALTER TABLE "users" ALTER COLUMN "role" SET DEFAULT 'member';
```

四件事一次看清：

- **默认值是列上的库属性**。改它不是改一行代码那么轻——它要的是一条 DDL，而这条 DDL 只有跑过 `npm run db:migrate` 才落到库里。
- 两句都写进**新文件**。旧的那份 `0001_core_tables.sql` 一个字节没动：四份文件的 sha256 前 12 位仍是 `6153a23bc13b`／`769be43ce241`／`30ceae56796b`／`cf7ba90d4e2b`。
- 两句里都没有 `--> statement-breakpoint`：一条语句不需要分句（上一站那条"标记数 = 语句数 − 1"在这里就是 0）。文件末尾也没有换行，字节数就等于句子长度——记账用的 hash 钉的是原样字节，多一个换行就对不上。
- **生成不碰库**。这两趟一条 SQL 都没发出去，所以库里那一列的默认值一直是 `'free'`，从头到尾没被动过。

两次实验都已还原：删掉那两份新 `.sql` 与对应的 `meta/0004_snapshot.json`、`meta/0005_snapshot.json`，再手工把 `meta/_journal.json` 里追加的两条删回去，`users.ts` 与备份逐字节比对相同（sha256 前 12 位 `66b20ac79095`）。还原后复跑上一站那条自检：

```text
dialect=postgresql  version=7  journal 条数=4
问题 0
```

这里留一条提醒：生成物留在目录里，就等于**给自己排了一次会真改库的动作**——下一次 `db:migrate` 会把那两句执行掉。所以试完就要撤干净，判据是 `问题 0` 加 `journal 条数=4`。

## 插入类型里只有 email 是必填

上面第 17 行那句报错里，TypeScript 把这张表的插入形状整个列出来了，读它就知道哪些列可以省：`id?`、`role?`、`createdAt?`、`bannedAt?`、`name?`、`githubId?` 都带问号，只有 `email` 是必填。

判据不是"有默认值才能省"，而是**`NOT NULL` 又没有 `DEFAULT` 的列才必填**：可空的列（`name`、`github_id`、`banned_at`）不写就是 `NULL`，也能省。

这条判据在真正的注册路径上就在用。`server/services/auth/verify.ts` 第 57 到 62 行：

```ts
export async function upsertUserByEmail (email: string): Promise<string> {
  const [inserted] = await useDb()
    .insert(users)
    .values({ email })
    .onConflictDoNothing()
    .returning({ id: users.id })
```

`values()` 里只有 `email` 一个键——号、角色、创建时间三项全部交出去，由库里的三个 `DEFAULT` 填。反过来看代码里哪里写了 `'free'`：`app/` 0 处、`server/` 0 处、`scripts/` 0 处，`db/` 里 2 处（枚举清单和那个 `.default('free')`），`test/` 里 10 处（端到端造数时用裸 SQL 显式写角色）。**应用层不出现这个角色值**，是这套默认值真的在库里生效的证据。

顺带一条别踩的坑：`shared/types/plan.ts` 里还有一个 `'free'`，那是订阅档位的名字（`'free' | 'pro'`），不是 `users.role`。两个词表撞在同一个字符串上，读代码时要分清。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 插入时报某列不能为 null | 那列是 `NOT NULL` 且没有 `DEFAULT`，库里没人填 | 要么给列加默认值并生成迁移，要么在应用里必填——先决定它归哪层 |
| 加了 `.default()`，库里的列却没变 | 生成只写文件，`db:migrate` 才执行 | 跑一次 `npm run db:migrate`（那是写库动作，先确认你连的是可丢弃的库） |
| 改默认值之后 `migrate` 报错说列已存在默认值之类 | 上一次生成的那份还在目录里，两次动作叠在一起 | `node scripts/check-migrations.mjs` 看当前排着几份，多余的那份连 `meta/` 里的记录一起撤 |
| `role` 写了新值，typecheck 红但库里也红 | 值域同时钉在 `pgEnum` 与库里的 `CREATE TYPE` | 加值要两步：改 `pgEnum`，再生成并执行那条 `ALTER TYPE`；只改一边必有一处不认 |
| 两条邮箱只差大小写却都建成功了 | `email` 不是 `citext`（换成 `text` 就会这样），或唯一约束不在 | 这一列必须是 `"citext"`，且看 `0001` 里那条 `UNIQUE("email")` 在不在 |

## 自己在自己的库上对一遍

本轮没有连库（连库与写库要先得到你的同意），所以下面这几条留给读者。前三条只读，第四条是改结构、要在可丢弃的库上跑：

1. 列的真实形状：

   ```sql
   select column_name, data_type, is_nullable, column_default
   from information_schema.columns
   where table_name = 'users'
   order by ordinal_position;
   ```

   七行，`column_default` 那一列应当出现 `gen_random_uuid()`、`'free'::user_role`、`now()` 三个值，其余四行是空的。
2. 两个唯一约束在不在：`select conname, contype from pg_constraint where conrelid = 'users'::regclass;`，预期 `users_pkey`、`users_email_unique`、`users_github_id_unique`。
3. 枚举真值清单：`select enumlabel from pg_enum e join pg_type t on t.oid = e.enumtypid where t.typname = 'user_role' order by enumsortorder;`——三行，顺序就是 `pgEnum` 里那个数组的顺序。
4. 默认值归哪层，最能体感的做法是不改代码直接写库：`insert into users (email) values ('case-test@example.com') returning id, role, created_at;` 一条都没给的值全由库补齐。跑这条之前先确认那个库是可以扔的。

## 这一站的边界

没做的事点名列出：没连库，所以上面四条读数是**预测的形状**，不是量出来的；库里那一列的 `column_default` 现值、"插入不在清单里的枚举值会被拒"那句原文，都还没取到；把整条迁移链拿到空库上重放跑到哪一句停，属这条线第 5 步；`github_id` 允许多条 `NULL` 这件事没验。类型那一趟是纯本地的，四行两对读数都在上文。

下一步 [751《用户表与会话表（2）：sessions 的外键为什么是 cascade》](/posts/8702668a938b19689e83ea3783a7ece5/)：只改 `onDelete` 那一个词，分两头看——类型那趟的报错逐字没变，SQL 这头生成出两条语句，这条规矩原来归数据库管。

## 本文依据

- 列的定义与生成的句子：本仓库 `db/tables/users.ts`、`db/tables/custom-types.ts`、`db/migrations/0001_core_tables.sql`
- `defaultRandom()` 在包里是 `default(sql\`gen_random_uuid()\`)`、`customType` 的 `dataType()` 返回什么迁移里就写什么：本机 `node_modules/drizzle-orm/pg-core/columns/uuid.js` 与 `columns/custom.d.ts`（同一条线前几站已引过行号）
- 类型检查要单独跑、`nuxt dev` 与 `nuxt build` 默认不查类型：<https://nuxt.com/docs/4.x/guide/concepts/typescript>
- 切段规则、生成不碰库、hash 钉原样字节、成对自检脚本：[749《开发库与第一份迁移（6）：这套文件怎么守住：入库、对账与类型护栏》](/posts/02c4422156fea45f01ecd620da8d2e9e/)，以及同一条线第 3、4 步
- 本轮读数：两次 `npm run db:generate` 的产物与字节数、`node scripts/check-migrations.mjs` 的输出、`npx nuxt typecheck` 的两行报错、`'free'` 在各目录的命中数
