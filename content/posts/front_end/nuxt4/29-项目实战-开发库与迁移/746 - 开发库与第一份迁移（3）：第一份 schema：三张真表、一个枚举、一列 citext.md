---
url: /posts/a198dbf106c4854fbc6e0062a5dbe0da/
title: 746 - 开发库与第一份迁移（3）：第一份 schema：三张真表、一个枚举、一列 citext
date: 2026-10-21T11:00:00+08:00
updated: 2026-10-21T11:00:00+08:00
author: cmdragon

summary:
  摘要：`db/tables/users.ts` 里那句 `citext('email')` 是怎么变成迁移文件里那行 `"email" "citext" NOT NULL` 的？这一站把第一份 schema 逐列对到生成的 SQL 上，再用一趟可撤销的实验看清 `npm run db:generate` 的输入与输出：加一列它会写出什么、撤掉之后又回到什么状态。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - Drizzle
  - 数据库迁移
  - PostgreSQL
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先看到的那六行

`npm run db:generate` 在什么都不改的时候，先把你的 schema 念一遍再告诉你它无事可做：

```text
6 tables
ping 3 columns 0 indexes 0 fks
users 7 columns 0 indexes 0 fks
sessions 5 columns 0 indexes 1 fks
login_tokens 6 columns 0 indexes 0 fks
audit_logs 8 columns 0 indexes 1 fks
meter_events 4 columns 0 indexes 1 fks

No schema changes, nothing to migrate 😴
```

这六行就是从 `db/schema.ts` 读出来的。那个文件只做两件事：把 `db/tables/` 下的表定义 re-export 出去，外加一张早期留下的 `ping`。它的注释写着"这个文件是结构应该长什么样的唯一入口，Nuxt 不扫描 `db/`，所以谁用谁 import"——服务端拿它的方式是一句 `import * as schema from '../../db/schema'`，这样 `db.select().from(users)` 才有类型可查。

## 三个词

**列类型**：Drizzle 里 `text('name')`、`uuid('id')` 这些构造器，每一个都对应一个写进迁移 SQL 的类型名。对得上就生成正常，对不上就得自己造（下一节那种）。

**`customType`**：包给你的口子，让你自己声明一个类型。它的 `dataType()` 返回什么字符串，迁移里那一列的类型名就写什么——这句是包内注释写的，不是猜测。

**`--> statement-breakpoint`**：生成物里的分句标记。有它，一份迁移才能被拆成一条一条执行；只有一条语句的新迁移文件里就不会出现它。

## 第一份真 schema 长什么样

`db/migrations/0001_core_tables.sql` 全文就 40 行，结构是"两行手工补的 ＋ 五条生成的"：

```sql [db/migrations/0001_core_tables.sql 的骨架]
create extension if not exists citext;
create extension if not exists pgcrypto;

CREATE TYPE "public"."user_role" AS ENUM('free', 'member', 'admin');--> statement-breakpoint
CREATE TABLE "users" ( ... );
--> statement-breakpoint
CREATE TABLE "sessions" ( ... );
--> statement-breakpoint
CREATE TABLE "login_tokens" ( ... );
--> statement-breakpoint
ALTER TABLE "sessions" ADD CONSTRAINT "sessions_user_id_users_id_fk"
  FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;
```

前面一站对账时数过：这一份 7 句里只有开头那两句扩展是可重放的，其余五句都会撞"已存在"。这一站看的是**它们和代码怎么一一对上**。

| `db/tables/` 里的写法 | 迁移 SQL 里出现的 |
| --- | --- |
| `pgEnum('user_role', ['free', 'member', 'admin'])` | `CREATE TYPE "public"."user_role" AS ENUM(...)`，而且排在 `CREATE TABLE "users"` 之前 |
| `citext('email').notNull().unique()` | `"email" "citext" NOT NULL` ＋ `CONSTRAINT "users_email_unique" UNIQUE("email")` |
| `uuid('id').primaryKey().defaultRandom()` | `"id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL` |
| `userRole('role').notNull().default('free')` | `"role" "user_role" DEFAULT 'free' NOT NULL` |
| `timestamp('created_at', { withTimezone: true }).notNull().defaultNow()` | `"created_at" timestamp with time zone DEFAULT now() NOT NULL` |
| `inet('created_ip').notNull()` | `"created_ip" "inet" NOT NULL` |
| `references(() => users.id, { onDelete: 'cascade' })` | 那句 `ALTER TABLE ... ADD CONSTRAINT ... ON DELETE cascade` |

顺序不是排版问题，是硬要求：`citext` 这个列类型由扩展提供、`gen_random_uuid()` 由 `pgcrypto` 提供，两者都得先存在，否则那两句 DDL 直接报"类型／函数不存在"。这就是为什么扩展行**手工**钉在文件最前面——生成器只管表、列、索引、外键，不认识扩展。

## 那一列 citext 为什么要自己造

Drizzle 的 Postgres 列构造器里没有 citext。这一条不靠记忆，靠包里查：

```text
'citext' in require('drizzle-orm/pg-core')   → false
'inet'、'customType'、'serial' 同样查        → true、true、true
grep -rli citext node_modules/drizzle-orm/pg-core/  → 命中 0 个文件
```

`pg-core/columns/` 下面一堆 `bigint`、`boolean`、`char` 之类的文件，唯独没有 citext。所以 `db/tables/custom-types.ts` 只有一个小函数：

```ts [db/tables/custom-types.ts 的核心]
export const citext = customType<{ data: string }>({
  dataType: () => 'citext',
})
```

`dataType()` 写什么，迁移里那一列就叫什么——所以上面那张表里 `"email" "citext"` 的来源其实是这两行代码。它带来的后果也写在同一个文件的注释里：**列类型名写成 `citext`，于是 `create extension if not exists citext` 变成建表的前置条件**。

顺带一条同类取证：`uuid('id').defaultRandom()` 在包里被实现成 `default(sql\`gen_random_uuid()\`)`（`pg-core/columns/uuid.js` 里那句注释写着"Adds `default gen_random_uuid()`"）。这不是 `uuid` 类型自带的能力，而是一个数据库函数——所以 `pgcrypto` 也必须先进那两行扩展。

## 加一列，生成器写出什么

本站做了一趟完整的可撤销实验：往 `login_tokens` 的字段表里加一行 `probeCol: text('probe_col')`，跑一次 `db:generate`。

```text
login_tokens 7 columns 0 indexes 0 fks          ← 原来是 6
[✓] Your SQL migration file ➜ db\migrations\0004_striped_polaris.sql 🚀
```

新文件全部内容就一句：

```sql [db/migrations/0004_striped_polaris.sql 全文]
ALTER TABLE "login_tokens" ADD COLUMN "probe_col" text;
```

三处细节值得看。第一，**没有 `--> statement-breakpoint`**——只有一条语句时不需要分句。第二，可空列就是 `ADD COLUMN "probe_col" text`，没有 `NOT NULL`，所以它不会要求你给已有行填值；反过来说，加一个 `notNull()` 又没默认值的列，这条语句会在有数据的表上失败，这是同一条规则的另一面（本轮没有试这一种，别当已证）。第三，除 `.sql` 之外还多了两样：`meta/0004_snapshot.json`（当时的完整结构记录）与 `_journal.json` 里追加的一条：

```json [journal 追加的那一条]
{ "idx": 4, "version": "7", "when": 1792236141458, "tag": "0004_striped_polaris", "breakpoints": true }
```

名字是生成器随机取的动物词——仓库里另外两份也是这么来的（`0002_previous_maddog`、`0003_legal_rattler`）。`drizzle-kit generate --help` 的十个旗标里有一个 `--name string`，不传就是随机名，所以 **tag 不能当语义用**，认顺序要看 `idx` 和文件名前缀。（传 `--name` 那趟本轮没跑。）

## 撤干净是可以验证的

把这行字段删掉、把新增的 `.sql` 与 `meta/0004_snapshot.json` 删掉、把 journal 追加的那条改回去，然后做两次检查：

```text
cmp db/migrations/meta/_journal.json <开工前的备份>   → 逐字节相同
journal sha256 前 12 位                              → 91a102b887d6（与基线一致）
四份 .sql 的 sha256 前 12 位                          → 与基线四行一致
再跑一次 db:generate                                  → No schema changes, nothing to migrate 😴
```

这组动作的意义：`db/migrations/` 是可以拿哈希当指纹管的地方。上一站说过，记账 hash 钉的是文件的**原样字节**；这一站的补充是——**生成这件事本身不碰库**，只写文件，所以可以放心做完再撤销。真要改库是 `db:migrate` 的职责，那是下一站。

## 那张 ping 表

`db/schema.ts` 还在导出 `ping`（三列：`serial` 主键、`note`、`created_at`），它是早期一次建表验证留下的。为什么留着：它的 `CREATE TABLE` 已经在 `0000_baseline.sql` 里了，现在删掉定义，下一份迁移就会多出一条 `DROP TABLE "ping"`——那是一次计划外的、会真删数据的动作。这一站只点名它，不删它；要清就得连带想清楚那份 `DROP` 归到哪一步。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 生成出的列类型名和期望不一样 | `customType` 的 `dataType()` 返回什么就写什么 | 改那个返回值，别去改生成出来的 `.sql` |
| 建表报"类型 citext 不存在" | 扩展行没在文件最前，或那个库根本没装扩展 | 确认开头两行 `create extension if not exists` 在位 |
| `defaultRandom()` 生成的库上没有 `gen_random_uuid()` | 它依赖 `pgcrypto` | 把扩展补上，或改用别的默认值 |
| 迁移文件叫 `0004_striped_polaris.sql` 看不懂改了啥 | 随机名 | 生成时传 `--name`；平时认 `idx` 与文件前缀 |
| 加一列变成删不掉 | 定义与库分叉：迁移里已经有那条 `ADD COLUMN` | 撤销要连 `.sql`、`meta/*_snapshot.json`、`_journal.json` 三处一起撤，再跑一次 generate 看是否回到无事可做 |
| 改了一份旧迁移的注释后行为变了 | 记账 hash 钉的是文件字节 | 已提交的迁移只追加、不修改 |

## 自己跑一遍

1. 先跑 `npm run db:generate`，把它念出来的那几行 `N tables / x columns` 记下来——那就是它眼中的结构。
2. 往任意一张表加一个可空列，再跑一次，读新 `.sql`：只有一条 `ALTER TABLE ... ADD COLUMN`，没有分句标记。
3. 打开 `meta/_journal.json`，看追加那条的 `idx`、`when`、`tag`、`breakpoints` 四个字段。
4. 全部撤掉，用 `cmp` 或哈希确认 journal 与四份 `.sql` 都回到原值，再跑一次确认回到 `No schema changes`。
5. 数一遍 `db/tables/` 里的构造器与 `node_modules/drizzle-orm/pg-core` 的导出，看看你要用的类型包里有没有——没有就照着 `custom-types.ts` 那一小段自己声明。

本轮的临时改动只在 `login_tokens` 上加了一行字段并生成出两份新文件，全部撤销并逐字节核对过；`db/schema.ts` 里改了一句过期注释（把某张表的来历指向了旧编排的编号）。改完复跑：`npx nuxt typecheck` **rc=0、输出 0 字节**，`npm run lint` **rc=0**（还是那条 `vue/no-v-html` 的 warning），`npm run test:unit` **25 文件 184 用例全过**。

下一步 [747《开发库与第一份迁移（4）：生成的 SQL 为什么还要手工补两行扩展》](/posts/eef88a3d2fbf9a7c4921f5ccf39cb539/)：加一个 citext 列再跑一次 generate，新文件里 `create extension` 一次都不出现——这层依赖没人记账，所以判据只能是整条链重放。

## 本文依据

- `customType` 与 `dataType()`（"迁移里用的数据库类型字符串表示"）、`pgEnum`／`pgTable`／`serial`／`inet`／`text`／`timestamp`／`uuid` 这些构造器的存在性：按本机安装的 drizzle-orm 0.45.4 的 `pg-core` 导出与 `columns/` 目录取证
- 结构定义与迁移生成的工作流（一份 schema 文件、`migrations/` 目录、`dbCredentials.url`）：来自 `drizzle.config.ts` 的四个配置项与 `drizzle-kit generate --help` 的旗标表
- `db/` 不属于 Nuxt 扫描范围、要靠 `typescript.nodeTsConfig.include` 才进类型检查（本轮未重跑那条红趟，留给这条线第 6 步）：`https://nuxt.com/docs/4.x/guide/concepts/typescript`
- 服务端拿到 schema 的方式（`import * as schema`）与 `db.select().from(...)` 的类型来源：`https://nuxt.com/docs/4.x/guide/concepts/server-engine`
- 迁移文件的幂等边界与"两个连接串名字"的分工：本仓库 `db/README.md` 的五条硬约定，上一站已逐条核到库上
