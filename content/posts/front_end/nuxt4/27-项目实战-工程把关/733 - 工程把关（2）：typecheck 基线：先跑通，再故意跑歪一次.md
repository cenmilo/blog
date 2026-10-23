---
url: /posts/dc1716e6512e76b7ec48cc6b52a10774/
title: 733 - 工程把关（2）：typecheck 基线：先跑通，再故意跑歪一次
date: 2026-10-19T10:00:00+08:00
updated: 2026-10-19T10:00:00+08:00
author: cmdragon

summary:
  摘要：类型检查这条命令最容易误会成"构建时已经做过了"。这一站先跑通拿到基线，再写一行 `const x: number = '这不是数字'` 故意跑歪：构建照样绿、typecheck 报 TS2322 且退出码是 2。然后把四份生成的 tsconfig 摊开看谁管哪一片，结果量出一个真实的洞——`scripts/` 目录里的 `.ts` 文件不在任何一份的收录范围里，lint 与类型检查两道都不响。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - typecheck
  - TypeScript
  - project references
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 屏幕上只有一行，而且不是它打的

先看两条命令跑完分别长什么样：

```text
$ npm run typecheck
> nuxt prepare && nuxt typecheck
│
◆  Types generated in .nuxt.
退出码 0        耗时 7116 ms

$ npm run build
…
└  ✨ Build complete!
退出码 0        耗时 25250 ms
```

`Types generated in .nuxt.` 那行是 `nuxt prepare` 打的，**`nuxt typecheck` 通过时一个字都不输出**（这一趟它的输出是 0 字节）。所以第一次跑的人会以为"是不是没跑"。

而那一行 `const x: number = '这不是数字'` 藏在 `app/utils/` 里的时候，上面这两条命令都是绿色的——**因为 Nuxt 默认不在 dev 和 build 里做类型检查**，官方给的原因是性能。要它检查，只有两条路：把 `vue-tsc` 与 `typescript` 装成开发依赖然后自己跑命令，或者在配置里把类型检查打开。本机装的是 `typescript 5.9.3`、`vue-tsc 3.3.13`。

## 故意跑歪一次，看它红

在这一句上面加一行赋值：

```ts [app/utils/ts-probe.ts]
export const probeValue: number = '这不是数字'
```

```text
$ npx nuxt typecheck
app/utils/ts-probe.ts(2,14): error TS2322: Type 'string' is not assignable to type 'number'.
退出码 2        耗时 6057 ms
```

三处要记住。报错格式是 `文件(行,列): error TS编号: 说明`，**不是** eslint 那种表格状输出；**退出码是 2**（eslint 的红是 1，`npm run check` 那种"非 0 就停"的判断对它同样有效）；同一次跑里，`npm run build` 依然返回 0 并打印 `✨ Build complete!`，日志里连一个 `TS` 编号都没有——这就是"构建绿不代表类型绿"的硬证据，也是 `typescript.typeCheck: true` 这个配置存在的意义（打开之后 dev 与 build 会顺带检查，代价是每次都要等类型）。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    typeCheck: true,
  },
})
```

这一位开了会怎样，值得量一次。把 `typeCheck: true` 加上、探针错误留在原地，再跑构建：

```text
$ npm run build          （typeCheck: true ＋ 那行错误）
app/utils/ts-probe.ts(2,14): error TS2322: Type 'string' is not assignable to type 'number'.
退出码 2        耗时 6899 ms          ← 没跑到打包就停了
```

**它是在构建最开始那一步把类型检查拦下来的**，6.9 秒就退出，比平时那趟 25 秒的构建短得多——所以"开了会不会拖慢失败"这个问题不存在，失败反而更早。真正要权衡的是**顺利的那条路**：同一份代码把错误删掉、开关留着再构建一次，

```text
$ npm run build          （typeCheck: true ＋ 无错误）
└  ✨ Build complete!
退出码 0        耗时 38264 ms          ← 不开时是 25250 ms
```

多出来的 13 秒就是这一位的代价。本项目最后没开它：`npm run check` 里 typecheck 是单独一项，失败照样拦，但每次构建不用付这 13 秒。

`nuxt typecheck` 自己的选项不多，全列在这里（默认那一栏空着的就是没有默认值）：

| 选项 | 默认 | 说明 |
| --- | --- | --- |
| `--cwd=<directory>` | | 指定项目根目录 |
| `--logLevel=<silent\|info\|verbose>` | | 构建期日志级别 |
| `--dotenv=<path>...` | | 要加载的 `.env` 路径，相对项目根；可以重复给，后面的文件优先 |
| `-e, --extends=<layer-name>...` | | 扩展某个 Nuxt 层 |
| `--checker=<vue-tsc\|golar>` | | 用哪个类型检查器；项目里如果有 `golar.config.*` 会让 Golar 变成默认，缺配置时还会替你写一份 |
| `-b, --build` | | 用 TypeScript project references 的 build mode 检查 |
| `--no-build` | | 不用 project references |

命令的入口参数只有一个 `ROOTDIR`（默认 `.`）。另外那条 note 值得知道：这条命令会把 `process.env.NODE_ENV` 设成 `production`，要盖它就在 `.env` 或命令行里给 `NODE_ENV`——这一轮没有专门验它。还有一个细节：`vue-tsc` 或 Golar 一个都没装时，命令会把对应包管理器的安装命令端给你，而不是静默失败。

## 覆盖面写在四份生成的配置里

Nuxt 生成的类型配置都放在 `.nuxt/` 里，跑 dev、build 或 prepare 都会重写。根目录那份 `tsconfig.json` 长这样：

```json [tsconfig.json]
{
  "files": [],
  "references": [
    { "path": "./.nuxt/tsconfig.app.json" },
    { "path": "./.nuxt/tsconfig.server.json" },
    { "path": "./.nuxt/tsconfig.shared.json" },
    { "path": "./.nuxt/tsconfig.node.json" }
  ]
}
```

只列引用、不列文件，这种形状官方叫 solution-style，讲命令那一节写明它会自动按 build mode 检查。四份被引用的配置各有各的地盘：

| 生成的配置 | include 条数 | 关键几项 | 明确排除 |
| --- | --- | --- | --- |
| `.nuxt/tsconfig.app.json` | 13 | `../app/**/*`、`../test/unit/**/*`、`../test/nuxt/**/*`、`../layers/*/app/**/*` | `../server`、`../.output`、`../nuxt.config.*`、`../modules/*.*` |
| `.nuxt/tsconfig.node.json` | 13（补洞后 14） | `../*.ts`、`../db/**/*`、`../test/e2e/**/*.ts`、`../nuxt.config.*` | `../app/**/*`、`../test/nuxt/**/*` |
| `.nuxt/tsconfig.server.json` | 6 | `../server/**/*`、`../shared/**/*.d.ts`、Nitro 自带的两份类型 | `../dist` |
| `.nuxt/tsconfig.shared.json` | 6 | `../shared/**/*`、`../modules/*/shared/**/*`、`../layers/*/shared/**/*` | 无 |

四份的 `compilerOptions` 里都是 `strict: true`、`noEmit: true`——官方那句"启用类型检查时默认打开 strict checks"就是落在这几份里，想临时关掉用 `typescript: { strict: false }`。还有一份 `.nuxt/tsconfig.json`（26 条 include）是为了向后兼容继续生成的，文档明写它将来会被移除，推荐用上面那四份。

两条纪律直接从这份结构里出来。第一，官方用 warning 说得很直：**不要直接改这些 tsconfig**，会被覆盖，要扩就通过 `nuxt.config`。"会被覆盖"不是猜的，把四份的修改时间记下来再跑一次 prepare：

```text
prepare 前  app 07:29:41 ／ node 07:27:26 ／ server 07:29:41 ／ shared 07:29:41
prepare 后  app 07:32:43 ／ node 07:32:43 ／ server 07:32:43 ／ shared 07:32:43
```

四份同一刻被重写。本项目就是靠 `typescript.tsConfig.include` 把 `test/unit` 加进 app 那份、靠 `nodeTsConfig.include` 把仓库根 `*.ts` 与 `db/**` 加进 node 那份（下一节还要往里加一位）。第二，官方讲项目引用的那节强调：**类型增补要放在对应的目录里**——给 app 上下文的增补文件就放 `app/`，server 的放 `server/`，两边共用的放 `shared/`；放在这些目录之外的增补，TypeScript 不会认。

至于 project references 换来的四个好处（官方列了：构建更快、IDE 更快、错误隔离、依赖显式），"更快"这一位在本项目量不出差别。三种跑法各来一次：

```text
npx nuxt typecheck              6393 ms
npx nuxt typecheck --no-build   6315 ms
npx nuxt typecheck --build      6145 ms
```

三个数在噪声范围内，谁也不比谁快。所以别把"开了引用就快"当成结论——这个仓库还没到那个规模。

## 量出来的洞：`scripts/` 里的 `.ts` 没人看

同一行错代码，换个位置放。这一趟把它写成两份，一份在 `app/utils/`、一份在 `scripts/`：

```text
$ npx nuxt typecheck            （两份都在，补洞前）
app/utils/ts-probe.ts(2,14): error TS2322: Type 'string' is not assignable to type 'number'.
退出码 2

$ npx eslint scripts/ts-probe.ts app/utils/ts-probe.ts
（两个文件都没问题）
退出码 0
```

`scripts/` 那份一次都没被报出来。原因不在代码，在收录范围：上面那张表里，**没有任何一份生成的 tsconfig 的 include 覆盖 `scripts/`**；而 lint 那一边——上一篇量过它按文件类型给不同规则（`.ts` 拿 82 条规则）——只判代码用法，不判类型。于是这个位置形成了两道闸门都不响的缝：一个 `.ts` 文件、看着就该有类型保护、结果谁都不看。

补法就是照官方那条路走，把目录加进配置，而不是手改生成的文件：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    nodeTsConfig: {
      // 补洞：scripts/ 下的 .ts 原本不在任何一份 include 里
      include: ['../*.ts', '../db/**/*', '../test/e2e/**/*.ts', '../scripts/**/*.ts'],
    },
  },
})
```

跑一次 `nuxt prepare` 再看生成物，`tsconfig.node.json` 的 include 从 13 条变成 14 条，最后那位正是 `../scripts/**/*.ts`。然后再跑一次：

```text
$ npx nuxt typecheck
app/utils/ts-probe.ts(2,14): error TS2322: Type 'string' is not assignable to type 'number'.
scripts/ts-probe.ts(2,14): error TS2322: Type 'string' is not assignable to type 'number'.
退出码 2
```

两份一起报，说明洞补上了。删掉探针文件，整套检查回到基线：

```text
npx nuxt typecheck    rc=0    6142 ms    输出 0 字节
npm run check         六项 rc=0：node 0ms ／ lint 3002ms ／ typecheck 6926ms ／
                               test:unit 1614ms ／ build 25190ms ／ test:nuxt 3880ms
                      Test Files 25 passed (25) ／ Tests 184 passed (184)
                      nuxt 层 Test Files 5 passed (5) ／ Tests 14 passed (14)
```

这条洞要留个尾巴说清：补的是 `scripts/**/*.ts`，**`scripts/` 里那几个 `.mjs` 仍然不在类型检查范围内**（它们压根不是 TS 文件，`allowJs` 也没开）。上一篇讲的那条缝——脚本里用服务端自动导入不会报——这次没解决，也不该指望 typecheck 解决；它的处置是"脚本里显式 import"，靠人守。

## 这一步的自查清单

1. 先确认基线：`npx nuxt typecheck` 退出码 0、输出为空。**别把"没有输出"当成"没有跑"**，判据用退出码。
2. 记住构建与类型检查是两件事：`npm run build` 绿不证明类型绿，除非开了 `typescript.typeCheck`。
3. 报错格式按 `文件(行,列): error TS编号` 读，退出码是 2；写聚合脚本时"非 0 就停"这条一样管得住它。
4. 想知道某个文件归谁管，去 `.nuxt/tsconfig.app.json`、`.node`、`.server`、`.shared` 四份里搜它的目录，不要猜。
5. 加目录用 `nuxt.config` 的 `typescript.tsConfig` / `nodeTsConfig`（app 那份与 node 那份），改完必须 `nuxt prepare` 才看得见效果。
6. 类型增补放对目录：`app/`、`server/`、`shared/` 三个上下文各管各的。

| 现象 | 先查什么 |
| --- | --- |
| typecheck 什么都没有输出 | 那就是通过；用退出码判 |
| 明明有类型错误，构建还是绿 | 默认行为，`typescript.typeCheck` 没开 |
| 某个目录里的错误一直没被报 | 那份目录是否在四份 include 里；不在就补配置再 prepare |
| 改了生成的 tsconfig 下一次又没了 | 正常，它们是 prepare 生成的，要改就改 `nuxt.config` |
| 自动导入的类型不生效 | 增补文件放错上下文了；按 app/server/shared 归位 |

下一步 [734《工程把关（3）：三层测试骨架：单测、Nuxt 运行时、端到端各测什么》](/posts/cf431b2686697c8882f4b3d3c4ab19a9/)：把测试拆成三层——纯 Node 的、要在 Nuxt 运行时里跑的、以及打真实产物的端到端，各测什么、各花多少时间、上一站那条洞在这一层怎么被用例钉住。

## 本文依据

- 默认 dev/build 不检查类型（性能原因）、装 `vue-tsc` 与 `typescript`、`npx nuxt typecheck`、`typescript.typeCheck`、四份 project references 配置各自管什么、legacy `.nuxt/tsconfig.json` 仍生成但将来移除、严格检查默认随类型检查开启且可用 `typescript.strict` 关、类型增补要放对上下文、不要直接改 tsconfig：`https://nuxt.com/docs/4.x/guide/concepts/typescript`
- `nuxt typecheck` 的入口参数与七个选项、Golar 检查器与 `golar.config.*`、solution-style 配置自动按 build mode、命令设 `NODE_ENV=production`、两者都没装时端出安装命令：`https://nuxt.com/docs/4.x/api/commands/typecheck`
- 自动生成类型位于 `.nuxt/`、由 dev/build/prepare 生成、包含对自动导入与 API 路由类型的引用与路径别名：`https://nuxt.com/docs/4.x/guide/concepts/typescript`、`https://nuxt.com/docs/4.x/directory-structure/nuxt`
- 本篇所有读数来自同一个仓库的连续多次运行：`typescript 5.9.3`、`vue-tsc 3.3.13`、Nuxt 4.5.2、Node v24.11.1；两份探针文件已删除，仓库留下的唯一改动是 `nuxt.config.ts` 里 `nodeTsConfig.include` 补的那一位
