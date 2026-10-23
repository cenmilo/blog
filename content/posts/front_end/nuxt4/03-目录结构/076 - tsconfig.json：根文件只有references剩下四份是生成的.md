---
url: /posts/0f80b036705b2827599bff219ec96cd3/
title: 076 - tsconfig.json：根文件只有 references，剩下四份是生成的
date: 2026-11-06T10:00:00+08:00
updated: 2026-11-06T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方对 tsconfig 这一节的安排很清楚：Nuxt 会自动生成多份 TypeScript 配置文件（.nuxt/tsconfig.app.json、.nuxt/tsconfig.server.json、.nuxt/tsconfig.node.json 与 .nuxt/tsconfig.shared.json），里面包含推荐的基础 TypeScript 配置、对自动导入与 API 路由类型的引用、路径别名等；而你的项目根应当只有一份 tsconfig.json，标准形态是 files 为空数组加上指向那四份生成配置的 references。官方在这份根文件下面直接用 warning 表态：不建议修改这个文件的内容，因为这样做可能覆盖 Nuxt 或其它模块依赖的重要设置，应改为通过 nuxt.config.ts 扩展。扩展入口在 typescript 这一节：typescript.tsConfig 为每一份生成配置设置共享的 compilerOptions，而 appTsConfig、sharedTsConfig、nodeTsConfig 与 serverTsConfig 分别定制对应那一份。官方还给了两类例外：lib、jsx 与 jsxImportSource 这类 DOM 与 Vue 专属选项只会应用到 tsconfig.app.json；而 types、paths 与 noEmit 由 Nuxt 按上下文管理（node、shared 与 server 三份刻意不产出任何内容也不扫描环境类型），写在 typescript.tsConfig 里不会改变它们。另一处已废弃：nitro.typescript.tsConfig 与 typescript.serverTsConfig 保持同步但已废弃，应改设后者。

categories:
  - 前端开发

tags:
  - nuxt4
  - typescript
  - tsconfig
  - project-references
  - 配置

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

036 那四查里第三查是根目录那份 `tsconfig.json` 被改过没有。这一篇就是那一查的完整版，这份文件本该长什么样、想改配置该去哪、以及哪些选项你写了也不生效。

分工，生成的四份加项目里的一份。官方原话，Nuxt 会自动生成多份 TypeScript 配置文件，`.nuxt/tsconfig.app.json`、`.nuxt/tsconfig.server.json`、`.nuxt/tsconfig.node.json` 与 `.nuxt/tsconfig.shared.json`，它们为你的项目包含推荐的基础 TypeScript 配置、对自动导入与 API 路由类型的引用、路径别名以及更多内容。你的 Nuxt 项目应当在根目录包含下面这份 `tsconfig.json` 文件。

```json [tsconfig.json]
{
  "files": [],
  "references": [
    {
      "path": "./.nuxt/tsconfig.app.json"
    },
    {
      "path": "./.nuxt/tsconfig.server.json"
    },
    {
      "path": "./.nuxt/tsconfig.shared.json"
    },
    {
      "path": "./.nuxt/tsconfig.node.json"
    }
  ]
}
```

看仔细，这份文件的 `files` 是空数组，它自己什么都不配置，只做四件事，指向四份生成配置。

这正是 035 那条 build mode 判定的依据，官方在那一节的原文是，一个 solution-style 的 `tsconfig.json`，只列出 `references` 的那一份，会自动以 build mode 进行类型检查。

所以这份文件怎么是空的、是不是坏了这个问题可以彻底放下，它就是设计成这样的。而且 035 说过，`typecheck` 会认出这个形态并自动切进 build mode，你手写了 `compilerOptions`，反而改变了它的行为。

那条 warning，不要改它。官方用 warning 写的，我们不推荐直接修改这个文件的内容，因为这样做可能覆盖 Nuxt 或其它模块依赖的重要设置。请改为通过 `nuxt.config.ts` 扩展它。

理由写得很具体，不是不优雅，而是这份配置里有其它模块也依赖的设置。

对照 052 那条同族的 warning，`.nuxt` 里任何文件都不要碰，因为 `nuxt dev` 会重建整个目录，这条是它的加强版，连项目根那份看起来是你自己的文件也别手改。

正确的入口，一份共享加四份分别定制。官方原话，你可以在 `nuxt.config.ts` 文件里自定义你 Nuxt 项目的 TypeScript 配置，用 `typescript.tsConfig` 为每一份上下文设置共享的 `compilerOptions`，并分别为每个上下文，app、shared、node 与 server，覆盖它们。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    // shared compiler options for every generated tsconfig
    tsConfig: {
      compilerOptions: {
        // ...
      },
    },
    // customize tsconfig.app.json
    appTsConfig: {
      // ...
    },
    // customize tsconfig.shared.json
    sharedTsConfig: {
      // ...
    },
    // customize tsconfig.node.json
    nodeTsConfig: {
      // ...
    },
    // customize tsconfig.server.json
    serverTsConfig: {
      // ...
    },
  },
})
```

五个键与四份生成配置一一对应，加上一个给所有份共享的 `tsConfig`。

| 键 | 落到哪份文件 |
| --- | --- |
| `tsConfig` | 每一份，多数 `compilerOptions` |
| `appTsConfig` | `.nuxt/tsconfig.app.json` |
| `sharedTsConfig` | `.nuxt/tsconfig.shared.json` |
| `nodeTsConfig` | `.nuxt/tsconfig.node.json` |
| `serverTsConfig` | `.nuxt/tsconfig.server.json` |

两类例外，这才是我配了但没生效的答案。官方在这个示例后面挂了一条 note，把上面那张表打破了两处，这两处都是 036 第四查的机制来源。

官方原话，多数 `compilerOptions` 与所有上下文共享，但有少数不是。像 `lib`、`jsx` 与 `jsxImportSource` 这些 DOM 与 Vue 专属的选项，只对你的应用代码有意义，因此它们只应用到 `tsconfig.app.json`。Nuxt 还按上下文管理 `types`、`paths` 与 `noEmit`，node、shared 与 server 这三份刻意不产出任何内容、也不扫描环境类型，所以在 `typescript.tsConfig` 里设置这些不会改变它们。当你需要覆盖它们时，请使用对应的按上下文选项。

拆成三条可执行结论。

`lib`、`jsx`、`jsxImportSource` 写在 `tsConfig` 里，服务端那份不会拿到。想让 `server/` 认某个东西，必须去 `serverTsConfig`。

`types`、`paths`、`noEmit` 写在 `tsConfig` 里是无效改动，官方原文就是不会改变它们。这不是 bug，是设计，那三份刻意不产出任何内容、也不扫描环境类型。

需要覆盖它们时，唯一入口是对应那份上下文的键。

第二类我建议你直接记成一句排错口诀，共享那份只管共享得动的键。

一个已废弃的旧入口。官方用 note 写的，`typescript.serverTsConfig` 扩展的是 `tsconfig.server.json`。`nitro.typescript.tsConfig` 与它保持同步，但已废弃，所以请改设 `typescript.serverTsConfig`。

这条正好在 065 讲过的 `nitro` 配置那一族里，065 提过可以用 `nuxt.config` 的 `nitro` 键直接设置 Nitro 配置。看到旧项目里写着 `nitro.typescript.tsConfig`，迁移动作就是搬去 `typescript.serverTsConfig`。

顺带把为什么分四份补完。031 抄过动机那句，Nuxt 使用 TypeScript project references 来提升类型检查性能并提供更好的 IDE 支持。

而这一节还有一条好处，官方列在 Benefits 里，更快的构建，TypeScript 可以跳过重建未发生变化的 project。

以及那份向后兼容的历史文件，官方也标了期限，出于向后兼容，Nuxt 仍然会生成 `.nuxt/tsconfig.json`。不过我们建议使用 TypeScript project references 配合那些新的配置文件以获得更好的类型安全与性能。这份遗留文件将在未来的某个版本被移除。

这条时间线值得知道，看到有人引用 `.nuxt/tsconfig.json`，单数、无后缀，那是旧写法，新代码要按四份走。

最后，测试目录也纳入这里。069 抄过的那段用的是同一个入口。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    tsConfig: {
      include: [
        // this path is relative to the generated .nuxt/tsconfig.json
        '../test/other-nuxt-context/**/*',
      ],
    },
  },
})
```

注意那句注释仍然是那句最关键的，`include` 里的路径相对于生成的 `.nuxt/tsconfig.json`，不是项目根。这一页教你改配置，那一页教你别把基准弄错。

## 自查

- [ ] 你项目根那份 `tsconfig.json` 的 `files` 应该是什么？只有 `references` 这一形态会触发 `typecheck` 的什么行为？
- [ ] 官方给的两条不要手改的理由分别是什么（根文件 / `.nuxt` 目录）？
- [ ] 想让 `server/` 里的代码认某个全局类型，该写哪个键？为什么不是 `typescript.tsConfig`？
- [ ] `types`、`paths`、`noEmit` 写在 `typescript.tsConfig` 会怎样？官方原话是什么？
- [ ] `nitro.typescript.tsConfig` 现在该换成什么？
- [ ] `.nuxt/tsconfig.json`（无后缀那份）还会存在多久、官方怎么定性它？

## 下一篇

077 讲 `.gitignore`：官方给的是一份至少包含的清单，四组里除了 `.nuxt` 与 `.output`，还有 `.data`、`.nitro`、`.cache` 与 `dist` 这几个名字，而它们在文档别处出现的场合，够你反推出生成物不止一处。
