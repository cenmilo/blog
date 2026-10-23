---
url: /posts/a67ad5f52bb89f6f205bf7cbde439073/
title: 739 - 目录边界（2）：形状只有一份：shared／types 两端同时消费
date: 2026-10-20T10:00:00+08:00
updated: 2026-10-20T10:00:00+08:00
author: cmdragon

summary:
  摘要：同一个共用类型，这个仓库里有三种写法：前端直接写名字、前端从 `#shared` 引、服务端用三层 `../` 引。哪一种才是必需的？这一站把类型放进六个不同位置，跑三趟类型检查，看每一侧免 import 到底认不认——顺带纠正一个容易照抄的说法：`app/types` 和 `server/types` 并不是写了就能直接用的地方。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 类型上下文
  - shared 目录
  - TypeScript
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 一个类型，三种写法

`app/composables/use-user.ts` 里第 8 行是这样写的：

```ts [app/composables/use-user.ts]
export function useUser () {
  return useState<CurrentUser | null>('currentUser', () => null)
}
```

`CurrentUser` 这个类型在这个文件里**没有 import**，也不报错。它的定义在 `shared/types/current-user.ts`。

同一个仓库里另外两处是另一种样子：

```ts [app/pages/docs/index.vue]
import type { DocSummary } from '#shared/types/doc'
```

```ts [server/services/auth/session.ts]
import type { CurrentUser } from '../../../shared/types/current-user'
```

三种写法同时存在，看起来像风格不统一。要判断哪种是必需的，只能把类型放到不同位置，看每一侧还认不认。这一节就把这件事一次做完。

## 先交代三个词

**类型上下文**：`nuxt prepare` 会在 `.nuxt/` 里生成四份 `tsconfig`——`app`、`server`、`shared`、`node`。前一份管 `app/` 里的代码，第二份管 `server/`，第三份管 `shared/`，最后一份管 `nuxt.config.ts` 和不在前三个目录里的文件。类型检查是按这四份分别跑的，所以"能不能认出来"这个问题永远要问清是哪一侧。

**自动导入注册表**：`.nuxt/imports.d.ts` 与 `.nuxt/types/` 下那几份 `.d.ts`。名字登记在里面，代码里才能不写 import 直接用。

**模块检测**：TypeScript 判断一个文件是"模块"还是"全局脚本"。规矩是老规矩——文件里没有 `import`/`export` 就当全局脚本，里面的 `interface` 于是成了全局名字。但生成的配置里有三份写死了 `"moduleDetection": "force"`，意思是**每个文件都按模块算**，这条路就被堵住了。

## 文档给的位置规矩

讲 shared 目录那一节的原话是：只有 `shared/utils/` 与 `shared/types/` 会被自动导入；单侧使用的类型可以放在靠近那一侧的地方——`app/types/` 在 Vue 应用里自动导入，`server/types/` 在 Nitro 服务端里自动导入。讲 server 目录那一节把第二句说得更满：放在 `~~/server/types/` 的类型"可以直接在 server 的路由、中间件、插件和工具函数里引用而不用 import"，并且给了这个例子：

```ts [server/types/todo.ts]
export interface Todo {
  id: string
  title: string
  completed: boolean
}
```

```ts [server/api/todos.get.ts]
export default defineEventHandler((): Todo[] => {
  return []
})
```

`api/nuxt-config` 的 `alias` 默认表里，`#shared` 指向 `<rootDir>/shared`，`#server` 指向 `<rootDir>/server`——这是显式导入时那两把别名的落点。

## 注册表里到底登记了谁

`nuxt prepare` 之后数四份文件（本轮读数，探针清干净之后的状态）：

```text
.nuxt/imports.d.ts               43 行，其中 4 行来自 shared/types
.nuxt/types/imports.d.ts        422 行，8 行来自 shared/types
.nuxt/types/shared-imports.d.ts  26 行，8 行来自 shared/types
.nuxt/types/nitro-imports.d.ts  216 行，8 行来自 shared/types
```

仓库里 `shared/types/` 下有四个文件、六个类型名：`CurrentUser`、`DocSummary`、`DocPage`、`MeterRow`、`MetricsSummary`、`PlanId`。这六个名字在四份文件里**各出现一次以上**，也就是说 app 侧和 server 侧都拿得到。`imports.d.ts` 里的样子：

```ts [来自 .nuxt/imports.d.ts 的真实两行]
export { CurrentUser } from '../shared/types/current-user';
export { DocSummary, DocPage } from '../shared/types/doc';
```

`types/imports.d.ts` 里还有另一套写法，它解释了"为什么不用 import"：

```ts [来自 .nuxt/types/imports.d.ts 的真实四行]
// for type re-export
declare global {
  export type { CurrentUser } from '../../shared/types/current-user'
  import('../../shared/types/current-user')
```

名字被放进了 `declare global`，所以整个应用上下文里直接写名字就认。同一份文件的第 3 行到第 200 行是另一批：函数与组合式函数的 `const xxx: typeof import(...)`。

四份注册表里，**跟 `app/types` 和 `server/types` 有关的行是 0**。这两个目录根本没被扫描登记。

## 六格矩阵：谁在哪一侧免 import

按位置放类型、按侧写消费方，跑 `npx nuxt typecheck`。这一趟是红的一趟，报错清单在下面，先把结论排成表：

| 类型放在 | app 侧免 import | server 侧免 import | 注册表里有它 |
| --- | --- | --- | --- |
| `shared/types/x.ts`，写 `export` | 认 | 认 | 四份都有 |
| `shared/types/nested/x.ts` | 不认 | 不认 | 没有 |
| `app/types/x.ts`，写 `export` | 不认 | 不认 | 没有 |
| `app/types/x.ts`，不写 `export` | 不认 | 不认 | 没有 |
| `app/types/x.ts`，用 `declare global` 包一层 | 认 | 不认 | 没有 |
| `server/types/x.ts`，写 `export` | 不认 | 不认 | 没有 |
| `server/types/x.ts`，不写 `export` | 不认 | 认 | 没有 |

同一趟报错清单里能对上表的每一行（去掉重复计数）：

```text
app/utils/probe-app-consume.ts(9,29): error TS2304: Cannot find name 'ProbeAppShape'.
app/utils/probe-app-consume.ts(10,28): error TS2304: Cannot find name 'ProbeAppAmbient'.
app/utils/probe-app-consume.ts(11,31): error TS2304: Cannot find name 'ProbeServerAmbient'.
app/utils/probe-app-consume.ts(14,9): error TS2741: Property 'extra' is missing in type
  'ProbeBothShape' but required in type 'ProbeCopiedShape'.
server/api/probe-shapes.get.ts(6,31): error TS2304: Cannot find name 'ProbeServerAmbient'.
server/api/probe-shapes.get.ts(7,32): error TS2304: Cannot find name 'ProbeServerShape'.
server/utils/probe-global-user.ts(1,48): error TS2304: Cannot find name 'ProbeGlobalShape'.
```

三件事直接从这份清单里读出来。`ProbeBothShape`（`shared/types/` 里带 `export` 的那个）一次都没出现，两侧都免 import。`ProbeAppShape` 与 `ProbeAppAmbient` 在 app 侧也报，所以 `app/types` 那句"自动导入"在 app 侧不成立——不带 `export` 也堵，因为 app 那份配置写死了模块检测。`ProbeServerAmbient` 只在 app 侧报、在 server 侧不报，`server/types` 那一侧确实免 import，但前提是**这个文件不写 `export`**。

文档给的 server 例子偏偏写了 `export`。

## 文档那个例子，照抄会红

把讲 server 目录那一节的例子原样落进一个只有 `nuxt.config.ts` 和 `app/app.vue` 的最小项目（`nuxt` 4.5.2、`typescript` 5.9.3、`vue-tsc` 3.3.13），`npx nuxt typecheck` 退出码 2，六行报错里这两行是主角：

```text
server/api/todos.get.ts(1,39): error TS2304: Cannot find name 'Todo'.
server/api/todos.get.ts(3,21): error TS2304: Cannot find name 'TodoServerAmbient'.
```

`Todo` 就是文档那个 `export interface Todo`，同一份文件里两处用到它、两处都报。同一个项目里，`shared/types/` 下的类型在这两个位置用到时一行错都没有。所以这不是项目大小的问题，也不是配置问题：**带 `export` 的类型只有放在 `shared/types/` 才两端都免 import**。

顺带一个会撞上的坑：这个最小项目原本没有根 `tsconfig.json`，`nuxt typecheck` 直接退出码 1：

```text
ERROR  Cannot find matching tsconfig.json in <项目根> or parent directories
```

补一个把四份生成配置引用进来的根配置（`"files": []` 加四条 `references`）之后才跑得动类型检查。类型检查依赖 `.nuxt/` 里的生成物，官方也明说不建议手改那几份文件，要扩展就写进 `nuxt.config.ts` 的 `typescript.tsConfig`。

## 那两种显式 import 是不是多余

`app/pages/docs/index.vue` 里那行 `import type { DocSummary } from '#shared/types/doc'` 属于"能写但不必需"。它必需的场景是类型不在扫描层里——比如 `shared/schemas/`（校验函数在那儿）和 `shared/types/nested/`（多一层子目录）。矩阵第五行那种 `declare global` 的写法能免 import，但代价是名字真的成了全局，`server` 侧看不见它：`server/utils/probe-global-user.ts(1,48)` 那条报错就是这么来的。

服务端用三层 `../` 的那种写法有它的原因：`#shared` 这个别名只在 Nuxt 的上下文里配好了。把四份生成的 `tsconfig` 的 `paths` 条目数出来是 **app 40 条、server 35 条、shared 39 条、node 19 条**，node 那一份里没有 `#shared`，也没有 `~`。这直接反映到测试层：

```ts [test/unit/：一行值导入]
import { formatCount } from '#shared/utils/format'
```

```text
vitest run  退出码 1
Error: Cannot find module '#shared/utils/format' imported from test/unit/probe-alias-value.test.ts
```

改成相对路径 `../../shared/utils/format` 同一份文件退出码 0。更要小心的是这一对：

```ts [test/unit/：一行只引类型的写法]
import type { PlanId } from '#shared/types/plan'
```

这一条**退出码 0**。`import type` 在编译期被擦掉，别名压根没被解析——绿灯不代表这条路通。值导入才是这一层的照妖镜。这也是为什么单测里引共用类型走相对路径、`test/nuxt/` 那一层才用 `#shared`（那一层跑在 Nuxt 上下文里）。

## 形状只有一份，防的是什么

`shared/types/metrics.ts` 里那两行：

```ts [shared/types/metrics.ts]
export interface MeterRow {
  kind: string
  n: number
}
```

服务端 `server/services/metrics/index.ts` 的签名是 `summarizeMeterRows(rows: MeterRow[]): MetricsSummary`，页面上读的是同一批字段名。把形状另抄一份会怎样？在 app 侧复制一份、多要一个 `extra` 字段：

```text
error TS2741: Property 'extra' is missing in type 'ProbeBothShape'
  but required in type 'ProbeCopiedShape'.
```

这一句就是"只有一份"的全部意义：loader 改了字段，抄一份的那一侧立刻报错；两处各写一套、字段名恰好都还在，才是最难查的那种 bug。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 类型放在 `app/types/` 里，页面直接写名字报 TS2304 | 这两个单侧目录不进注册表 | 挪到 `shared/types/`，或者显式 import |
| `server/types/` 里带 `export` 的类型在 handler 里报 TS2304 | 文档那个例子在这一版跑不通 | 同上，或者去掉 `export`——但那是运气不是规则 |
| `shared/types/` 下面再开一层子目录就认不出 | 只扫直接一层 | 用 `#shared/types/子目录/文件` 显式引 |
| 同一个报错出现两遍 | 四份配置各自扫过，交叉文件被数两次 | 按文件加行号去重再看条数 |
| 单测里 `#shared` 报 Cannot find module | node 那一份 `paths` 里没有这组别名 | 单测走相对路径，`test/nuxt/` 才用别名 |
| 单测里 `import type` 从 `#shared` 能过 | 只引类型会被擦掉，没解析过别名 | 别把它当成别名可用的证据 |
| `nuxt typecheck` 报 Cannot find matching tsconfig.json | 项目根上没有 `tsconfig.json` | 补一份引用四份生成配置的根文件 |

## 自己跑一遍

1. 在 `shared/types/` 放一个带 `export` 的接口，在 `app/utils/` 和 `server/api/` 各写一处直接用它，跑 `npx nuxt typecheck`——退出码 0。
2. 把那个文件挪进 `shared/types/nested/`，同一趟立刻变成两条 `Cannot find name`；加一行 `import type ... from '#shared/types/nested/...'` 又回去。
3. 再把它挪到 `app/types/`——app 侧也报。这就是文档那句"`app/types/` 在 Vue 应用里自动导入"的实际样子。
4. 在 `server/types/` 建一个带 `export` 的接口，在 handler 里直接用——照样报。去掉 `export` 再试，server 侧通了，app 侧仍然不通。
5. 全部探针删干净，重跑 `nuxt prepare` 和 `nuxt typecheck`，退出码回到 0、输出 0 字节。

这一站的六个探针文件与两个测试文件都已删除，`.nuxt/` 里搜 `probe` 命中 0；删完重跑类型检查退出码 0、输出 0 字节，`npm run lint` 退出码 0（还挂着那条 `vue/no-v-html` 的 warning），单测 25 个文件 184 条用例全过。

下一步 [740《目录边界（3）：pages 与 layouts：谁被扫描、注册成什么》](/posts/c32e3025857237f0fc287b8d67de1040/)：页面文件名怎么变成路由记录、布局文件名怎么变成一张类型表里的键，以及写错布局名时那四道类型护栏各自挡在哪。

## 本文依据

- 只有 `shared/utils/` 与 `shared/types/` 被自动导入、嵌套子目录不算、`import type` 会被擦除、共用类型放 `shared/types/`、`app/types/` 与 `server/types/` 各自只在一侧自动导入、`#shared` 别名：`https://nuxt.com/docs/4.x/directory-structure/shared`
- `#server` 别名只能在 `server/` 侧用、`server/types/` 只扫直接一层、文档给的 `export interface Todo` 例子：`https://nuxt.com/docs/4.x/directory-structure/server`
- 类型可以像工具函数一样自动导入、`app/types/` 的位置建议：`https://nuxt.com/docs/4.x/directory-structure/app/utils`
- 四份类型上下文各管什么、不要手改生成的 `tsconfig.json`、扩展走 `typescript.tsConfig`：`https://nuxt.com/docs/4.x/guide/concepts/typescript`
- `alias` 默认表里 `#shared`、`#server`、`~`、`~~` 的指向，以及别名会自动进生成的配置：`https://nuxt.com/docs/4.x/api/nuxt-config`
