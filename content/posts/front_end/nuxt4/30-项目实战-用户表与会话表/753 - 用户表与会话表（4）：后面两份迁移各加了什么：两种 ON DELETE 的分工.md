---
url: /posts/85041fb63edb1bf9267bd29d431935a2/
title: 753 - 用户表与会话表（4）：后面两份迁移各加了什么：两种 ON DELETE 的分工
date: 2026-10-22T12:00:00+08:00
updated: 2026-10-22T12:00:00+08:00
author: cmdragon

summary:
  摘要：`0002` 建审计表、`0003` 建计量表，两份文件的形状一模一样——一段建表、一个分句标记、一段挂约束。区别只在那两个词：一张用 set null，一张用 cascade。这一站读这两句 SQL，再用类型和两次生成量出这件事归谁管，以及一个真正会让人翻车的组合：把 set null 写在一条 NOT NULL 的列上，两道闸门都不拦。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 外键
  - 数据表设计
  - Drizzle
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 两份文件，形状一样，只差两个词

`db/migrations/0002_previous_maddog.sql` 十一行、440 字节，`0003_legal_rattler.sql` 七行、381 字节。两份的结构完全同构：**一段 `CREATE TABLE`，一个 `--> statement-breakpoint`，一段 `ALTER TABLE … ADD CONSTRAINT`**。

前面几站数过那条规律（分句标记数 = 语句数 − 1），这两份就是最干净的两个样本：各 1 个标记、各 2 段。为什么建表要分两段执行？因为生成器把表定义和约束分开写——表先存在，约束再挂上去。

两份最后那一句，是这一站的全部内容：

```sql
-- 0002 的第二段
ALTER TABLE "audit_logs" ADD CONSTRAINT "audit_logs_actor_users_id_fk" FOREIGN KEY ("actor") REFERENCES "public"."users"("id") ON DELETE set null ON UPDATE no action;
-- 0003 的第二段
ALTER TABLE "meter_events" ADD CONSTRAINT "meter_events_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;
```

## 各自加了什么

| | `audit_logs`（0002） | `meter_events`（0003） |
| --- | --- | --- |
| 主键 | `"id" bigserial PRIMARY KEY NOT NULL` | `"id" bigserial PRIMARY KEY NOT NULL` |
| 与人相关的那一列 | `"actor" uuid`——可空 | `"user_id" uuid NOT NULL`——必填 |
| 删除规则 | `ON DELETE set null` | `ON DELETE cascade` |
| 内容列 | `"action" text NOT NULL`、`"target" text NOT NULL` | `"kind" text NOT NULL` |
| 数字列 | `"count" bigint NOT NULL` | 没有 |
| 结构化列 | `"meta" jsonb` | 没有 |
| 时间列 | `"ts" timestamp with time zone DEFAULT now() NOT NULL` | `"occurred_at" timestamp with time zone DEFAULT now() NOT NULL` |
| 另一个可空列 | `"request_id" text` | 没有 |

两处选择都不是随手来的，`db/tables/` 里那两份注释就是正面材料：审计行要回答"是谁动了谁的什么"，**人删了痕迹必须留着**，所以 `actor` 可空加 `set null`，读出来那个 `null` 就代表"这个账号已经不在了"；计量行回答"这个人做过几次什么"，是这个人自己的东西，**账号没了留一堆对不上号的匿名计数**没意义，所以 `cascade`。

`meter_events.ts` 的注释里还专门写了一句"这条与 `audit_logs.actor` 的选择正好相反，别顺手照抄"——这一站的存在理由就是那句提醒：两种规则确实容易抄错，而抄错的后果两头都看不见。

顺带两列值得一提。`bigserial` 在 `db/tables/audit-logs.ts` 里写的是 `bigint('count', { mode: 'number' })`——不写 `mode` 读回来是字符串，写了才是 `number`；`meta` 那列是 `jsonb` 加了 `.$type<Record<string, string | number | boolean>>()`，所以 JSON 列也有类型，代价是这个泛型只到你写的那一层，嵌套对象它不看（这条边界本文没验）。

## 类型那一趟：这件事落到类型上长什么样

上一站欠一条读数，这一站补上：`set null` 那条规矩在类型里是什么形状。探针六段声明（跑完就删），`npx nuxt typecheck` 报三行、`rc=2`：

```text
db/probe-ondelete-types.ts(15,14): error TS2322: Type 'null' is not assignable to type 'string'.
db/probe-ondelete-types.ts(21,14): error TS2741: Property 'count' is missing in type '{ action: string; target: string; }' but required in type '{ target: string; action: string; count: number; meta?: Record<string, string | number | boolean> | null | undefined; id?: number | undefined; ts?: Date | undefined; actor?: string | ... 1 more ... | undefined; requestId?: string | ... 1 more ... | undefined; }'.
db/probe-ondelete-types.ts(24,14): error TS2741: Property 'userId' is missing in type '{ kind: string; occurredAt: Date; }' but required in type '{ userId: string; kind: string; id?: number | undefined; occurredAt?: Date | undefined; }'.
```

三件事当场定死：

- 第 15 行报错的是 `MeterRow['userId'] = null` ⇒ **cascade 那张表的这一列读出来是 `string`，永远不会是 null**。
- 而另外两行**没报错**：`AuditRow['actor'] = null` 和 `= '11111…'` 都通过 ⇒ **`set null` 那一列在类型里是 `string | null`**。这就是"人删了、痕迹还在"这件事在代码里的形状：读审计行的时候，那一位得当可空处理，`actor` 不能直接喂给一个只接 `string` 的函数。
- 插入形状：审计行必填 `target`、`action`、`count`（`actor`、`ts`、`id`、`requestId`、`meta` 都能省），计量行必填 `userId`、`kind`。

## 两次生成：一次改规则，一次改出一个矛盾

第一趟把 `audit_logs.actor` 的 `set null` 改成 `cascade`。生成出 `0004_lazy_deathstrike.sql`，**263 字节、1 个分句标记、两段**：

```sql
ALTER TABLE "audit_logs" DROP CONSTRAINT "audit_logs_actor_users_id_fk";
--> statement-breakpoint
ALTER TABLE "audit_logs" ADD CONSTRAINT "audit_logs_actor_users_id_fk" FOREIGN KEY ("actor") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;
```

这与[上一站](/posts/8702668a938b19689e83ea3783a7ece5/)在 `sessions` 上量到的形状一致：**改一条外键 = 先删约束再建约束**，名字复用。（两份都恰好 263 字节是表名与列名各短了一点凑出来的，别当规律用。）

第二趟是真有教学价值的那一趟。把 `meter_events.user_id` 的 `cascade` 改成 `set null`——**这一列是 `NOT NULL` 的**，这是个语义上互相打架的组合。结果：

```text
meter_events 4 columns 0 indexes 1 fks
文件 0004_tense_bruce_banner.sql 字节 278 断点 1
```

生成出的第二段写着 `ON DELETE set null`，而第一段的表定义里那一列还是 `NOT NULL`。**生成器照做，不警告、不报错，退出码 0**；再跑一次类型检查，探针的三行报错与第一趟逐字相同，`meter_events.ts` 自己**零条报错**。

所以这个矛盾组合在这台机器上能一路走到"待执行"状态：库里的承诺（删用户时把这一列置成 null）和类型的承诺（这一列读出来是 `string`，不可能是 null）**正面冲突，而拦得住它的一道闸门在数据库里**——那条 DDL 执行时会不会被拒、还是等到真删用户那天才炸，本轮没有连库，所以不下结论，交给读者在可丢弃的库上跑一次（下面第 3 条）。这一站的结论就压在这句上：**外键规则这件事，生成器和类型都不给你兜底。**

两趟实验都已还原并逐字节核对：`audit-logs.ts` 回到 `dceca6a39e18`、`meter-events.ts` 回到 `5057ebcb0a30`，两份新 `.sql` 与对应的 `meta/000N_snapshot.json` 删净、journal 回到 `91a102b887d6`／四条，自检脚本 `问题 0`，再跑一次生成回到 `No schema changes, nothing to migrate`。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 删用户报"还有别的行引用它" | 那张表挂的是 `no action`／`restrict`，数据库不替你处理 | 要么应用先删，要么把规则改成符合归属关系的那一种 |
| 审计行一夜之间全没了 | `actor` 被写成了 `cascade` | 追责用的表用可空列加 `set null`；痕迹不能跟着人走 |
| 仪表盘上留着一堆对不上号的计数 | 计量行用了 `set null` 或没挂外键 | 属于"这个人的东西"就用 `cascade` |
| 读审计行时 `actor` 被判成非空，编译过不去或运行时炸 | 类型里它是 `string \| null`（本轮第 1、2 段探针） | 写判空分支，别假设它总有值 |
| 把 `set null` 加在 `NOT NULL` 列上，一路没人拦 | 生成器只管语法、类型只看 `.notNull()`，两边都不知道这两句会打架 | 改完外键规则，手工读一遍生成的那两段 SQL；这条只能靠人 |

## 自己在自己的库上验

本轮没有连库，下面三条都是待跑动作，第三条会真删数据、务必用可丢弃的库：

1. 两张表的规则确实不同：`select conname, confdeltype from pg_constraint where conname in ('audit_logs_actor_users_id_fk','meter_events_user_id_users_id_fk');` 两行的 `confdeltype` 是两个不同字符（具体对应关系本文不断言，跑出来一眼能对上）。
2. 那句矛盾 DDL 到底答不答应：在一个测试库上手工执行 `ALTER TABLE … ADD CONSTRAINT … ON DELETE set null …`，指向一列 `NOT NULL` 的列。
3. 分工的体感：给一个用户各写一行审计与一行计量，然后删掉那个用户——预期审计行还在而 `actor` 是 null，计量行没了。这两条判据合起来才是"两种规则各有用处"。

## 这一站的边界

一次库连接都没建立，所以上面三条没有读数；两份迁移的字节数、断点数、生成的句子、三行类型报错是本地跑出来的。空库整链重放（拿到一个干净实例，看这条链跑到哪一句会停）是下一站。

下一步 [754《用户表与会话表（5）：整条链拿到空库上会跑到哪一句停》](/posts/3da8456c0626a0d75825dc53ad632766/)：不再改规则，把四份文件当一条链走一遍——十段句子各需要什么、谁在什么时候把它造出来，顺带量出那套自检脚本管到哪儿、管不到哪儿。

## 本文依据

- 两份迁移的全文与断点位置：本仓库 `db/migrations/0002_previous_maddog.sql`、`db/migrations/0003_legal_rattler.sql`
- 两种规则的出处与理由：`db/tables/audit-logs.ts`、`db/tables/meter-events.ts` 的文件头注释（含"别顺手照抄"那句）
- 类型读数：`npx nuxt typecheck` 对临时探针 `db/probe-ondelete-types.ts` 输出的三行（该文件已删）
- 改一条外键生成两条语句、外键规则不进类型：[751《用户表与会话表（2）：sessions 的外键为什么是 cascade》](/posts/8702668a938b19689e83ea3783a7ece5/)；分句标记与语句数关系、生成不碰库：[749](/posts/02c4422156fea45f01ecd620da8d2e9e/)
- 类型检查要单独跑、`nuxt dev` 与 `nuxt build` 默认不查类型：<https://nuxt.com/docs/4.x/guide/concepts/typescript>
- 本轮读数：两次 `npm run db:generate` 的产物字节数与段数、两趟 typecheck 的报错对比、还原后的 `问题 0` 与 `No schema changes`
