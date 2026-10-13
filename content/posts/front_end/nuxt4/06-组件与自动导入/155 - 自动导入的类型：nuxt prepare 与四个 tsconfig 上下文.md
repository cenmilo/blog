---
url: /posts/5b0046a23aac588f4fd0a143339b0ebc/
title: 155 - 自动导入的类型：nuxt prepare 与四个 tsconfig 上下文
date: 2026-12-02T11:00:00+08:00
updated: 2026-12-02T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：前面几篇里"自动导入还保留类型"这件事我一直当空气，这一篇把它落到文件上。官方原话："Nuxt projects rely on **auto-generated types** to work properly. **These types are stored in the `.nuxt` directory and are generated when you run the dev server or build your application. You can also generate these files manually by running `nuxt prepare`.**"。`prepare` 命令自己的定位是："**The prepare command creates a .nuxt directory in your application and generates types. This can be useful in a CI environment or as a `postinstall` command in your `package.json`.**"，并且有一条容易被忽略的 note："**This command sets `process.env.NODE_ENV` to `production`.**"。生成的不止一份：Nuxt 用 TypeScript project references，"**When you run `nuxt dev`, `nuxt build` or `nuxt prepare`, Nuxt will generate multiple `tsconfig.json` files for different parts of your application.**"，四个上下文的官方定义是——`.nuxt/tsconfig.app.json`"**Configuration for your application code within the `app/` directory**"、`.nuxt/tsconfig.node.json`"**Configuration for your `nuxt.config.ts` and files outside the other contexts**"、`.nuxt/tsconfig.server.json`"**Configuration for server-side code (when applicable)**"、`.nuxt/tsconfig.shared.json`"**For code shared between app and server contexts (like types and non-environment specific utilities)**"。根目录的 `tsconfig.json` 应该是 `{"files": [], "references": [app/server/shared/node 四条 path]}`。两条"别乱动"的警告各在一页出现："**it is not recommended to modify your `tsconfig.json` file directly, as doing so could overwrite important settings**" 与 "**We do not recommend modifying the contents of this file directly... Instead, extend it via `nuxt.config.ts`.**"，正解是 `typescript.tsConfig`（所有上下文共享）加四个按上下文的键 `appTsConfig`/`sharedTsConfig`/`nodeTsConfig`/`serverTsConfig`，并注明"**Most `compilerOptions` set in `typescript.tsConfig` are shared with every context, but a few are not. DOM- and Vue-specific options (such as `lib`, `jsx` and `jsxImportSource`) only make sense for your application code**"。最实用的一句是关于类型增强的：**"TypeScript will not recognize augmentations placed outside these directories unless they are explicitly included in the appropriate context."**

categories:
  - 前端开发

tags:
  - nuxt4
  - TypeScript
  - tsconfig
  - prepare
  - 类型检查

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


150 引过那句承诺——自动导入"preserves typings"。但它到底靠什么保住类型，以及为什么有时"我的 `interface` 增强明明写了却不生效"，答案都在生成文件与四个上下文里。

## 类型是生成的文件，不是编辑器的魔法

> Nuxt projects rely on auto-generated types to work properly. These types are stored in the `.nuxt` directory and are **generated when you run the dev server or build your application**. You can also generate these files manually by running **`nuxt prepare`**.

这三句决定了一件事：**没跑过 dev/build/prepare，类型就不存在**。刚 clone 的仓库直接开编辑器会满屏红线，不是配置错了，是那批文件还没生成。

`prepare` 那一节给的场景与一句重要提醒：

> The prepare command creates a `.nuxt` directory in your application and generates types. This can be useful in a CI environment or as a `postinstall` command in your `package.json`.

> This command sets `process.env.NODE_ENV` to `production`.

**第二句对 CI 有影响**：prepare 是以"生产"环境跑的。它接受的参数与选项：

| 项 | 官方说明 |
| --- | --- |
| `ROOTDIR` | "The root directory of your Nuxt project (default: `.`)" |
| `--cwd=<directory>` | "Specify the root directory of your Nuxt project" |
| `--dotenv=<path>...` | "Path to `.env` file to load, relative to the root directory. **Can be repeated, with later files taking precedence.**" |
| `--logLevel=<silent\|info\|verbose>` | "Specify build-time log level" |
| `--envName=<environment>` | "The environment to use when resolving configuration overrides (**default is `production` when building, and `development` when running the dev server**)" |
| `-e, --extends=<layer-name>...` | "Extend from a Nuxt layer" |

`--dotenv` 那句"later files taking precedence"与 `--envName` 的默认值两行，是配置类脚本里最容易写反的两处。

## 一个项目里有四份 tsconfig

官方给的原因很直白：

> Nuxt uses TypeScript project references to improve type-checking performance and provide better IDE support. This feature allows TypeScript to break up your codebase into smaller, more manageable pieces.

> When you run `nuxt dev`, `nuxt build` or `nuxt prepare`, Nuxt will generate multiple `tsconfig.json` files for different parts of your application.

四个文件与它们的上下文：

| 生成的文件 | 官方定义 | 谁在里面 |
| --- | --- | --- |
| `.nuxt/tsconfig.app.json` | Configuration for your application code **within the `app/` directory** | `app/` 下的页面、组件、composables、plugins… |
| `.nuxt/tsconfig.node.json` | Configuration for your **`nuxt.config.ts` and files outside the other contexts** | 配置文件与"哪儿都不属于"的代码 |
| `.nuxt/tsconfig.server.json` | Configuration for **server-side code** (when applicable) | `server/` |
| `.nuxt/tsconfig.shared.json` | For **code shared between app and server contexts** (like types and non-environment specific utilities) | `shared/` |

官方给的项目根 `tsconfig.json` 就只是四条引用：

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

**`"files": []` 是刻意的**：根文件不直接收文件，只挂四个子项目。

还有一份遗留的：

> For backward compatibility, Nuxt still generates `.nuxt/tsconfig.json`. However, we recommend using TypeScript project references with the new configuration files (`.nuxt/tsconfig.app.json`, `.nuxt/tsconfig.server.json`, etc.) for better type safety and performance. **This legacy file will be removed in a future version of Nuxt.**

（这正好接上 087 那条：`buildDir` 与 `typesDir` 被故意分开配，就是为了让根 `tsconfig.json` 里这四条 `references` 一直有效。）

项目 references 的四条好处，官方原文列了：

> - **Faster builds**: TypeScript can skip rebuilding unchanged projects
> - **Better IDE performance**: Your IDE can provide faster IntelliSense and error checking
> - **Isolated compilation**: Errors in one part of your application don't prevent compilation of other parts
> - **Clearer dependency management**: Each project explicitly declares its dependencies

第三条我认为是这批里最能改变排错习惯的：**"app 里报错不会挡住 server 编译"**，反过来说——你看到某个上下文的报错"消失了"，也可能只是那份子项目没被检查。

## 增强必须写在对的目录里

这一节回答"我加了 `declare module` 却没生效"：

> Since the project is divided into **multiple type contexts**, it's important to **augment types within the correct context** to ensure they're properly recognized. **TypeScript will not recognize augmentations placed outside these directories unless they are explicitly included in the appropriate context.**

官方给的三条落点：

> For example, if you want to augment types for the `app` context, the augmentation file should be placed in the `app/` directory.
>
> Similarly:
> - For the `server` context, place the augmentation file in the `server/` directory.
> - For types that are **shared between the app and server**, place the file in the `shared/` directory.

把这张表叠到前面几篇的增强写法上：

| 你增强的东西 | 增强的接口 | 文件该放哪 |
| --- | --- | --- |
| 页面里 `definePageMeta` 的自定义字段 | `#app` 的 `PageMeta`（143） | `app/` 下 |
| 钩子/模块里 `page.meta` 的自定义字段 | `@nuxt/schema` 的 `NuxtPageMeta`（143） | 按它被谁读取来放；跨两侧就放 `shared/` |
| `useAppConfig()` 在 app 侧的类型 | `AppConfig`（108） | `app/` 下 |
| `useAppConfig()` 在 app 之外的类型 | `SharedAppConfig`（108、151） | `shared/` 或对应上下文 |

**注意那句 "unless they are explicitly included in the appropriate context"**：放在外面并非绝对不行，而是要显式纳入某个上下文——官方把这条路指向了模块作者指南的 "extend typescript config" 一节（`guide/modules/recipes-advanced`）。应用项目里我不走这条路，直接按表放对目录更省事。

## 别手改，用 `typescript` 那五个键

两节各有一条警告，方向一致：

> Nuxt relies on this configuration, and Nuxt modules can extend it as well. For this reason, **it is not recommended to modify your `tsconfig.json` file directly**, as doing so could overwrite important settings. Instead, extend it via `nuxt.config.ts`.

> We do not recommend modifying the contents of this file directly, as doing so could overwrite important settings that Nuxt or other modules rely on. Instead, extend it via `nuxt.config.ts`.

正解的形状官方给全了：

> You can customize the TypeScript configuration of your Nuxt project in the `nuxt.config.ts` file: **set shared `compilerOptions` for every context at once with `typescript.tsConfig`, and override them for each context (`app`, `shared`, `node`, and `server`) individually.**

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

一条例外官方用 note 写了出来，别指望 `tsConfig` 真的"共享一切"：

> Most `compilerOptions` set in `typescript.tsConfig` are shared with every context, but a few are not. **DOM- and Vue-specific options (such as `lib`, `jsx` and `jsxImportSource`) only make sense for your application code, so they are applied to `tsconfig.app.json` alone.** Nuxt also manages `types`, `paths` and `noEmit` per context (the `node`, `shared` and `server` configs deliberately emit nothing and scan no ambient types), so **setting these in `typescript.tsConfig` will not change them.** Use the matching per-context option (`appTsConfig`, `nodeTsConfig`, `sharedTsConfig` or `serverTsConfig`) when you need to override them.

这段里有两组"例外"，方向刚好相反：

| 选项 | 在 `tsConfig` 里的行为 | 正确去处 |
| --- | --- | --- |
| `lib`、`jsx`、`jsxImportSource` | 不会共享给四个上下文，**只作用于 `tsconfig.app.json`** | 想改就改 `appTsConfig` |
| `types`、`paths`、`noEmit` | **你在 `tsConfig` 里设了也不会生效**（Nuxt 按上下文自己管；`node`/`shared`/`server` 三份故意不 emit、也不扫环境类型） | 用对应上下文的键覆盖 |

第二组对这一篇的主题尤其相关：**`paths` 就是那些别名（`#imports`、`~`、`#build`）的所在**，官方明说它是"Nuxt manages per context"的一项。也就是说想动别名不能靠 `typescript.tsConfig`，得用按上下文的键——这也解释了为什么 154 那批别名能在四个上下文里各自成立。

还有一条同族的替换关系，官方一句话讲完：

> `typescript.serverTsConfig` extends `tsconfig.server.json`. **`nitro.typescript.tsConfig` is kept in sync with it but is deprecated, so set `typescript.serverTsConfig` instead.**

## 类型检查默认根本不开

这一句能解释"我删了个字段却没报错"：

> By default, Nuxt **doesn't check types** when you run `nuxt dev` or `nuxt build`, **for performance reasons**.

要开有两条路，第一步是装包：

> To enable type-checking at build or development time, install `vue-tsc` and `typescript` as development dependency:

```bash [npm]
npm install --save-dev vue-tsc typescript
```

```bash [Terminal]
npx nuxt typecheck
```

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    typeCheck: true,
  },
})
```

还有一条连锁，官方写得很具体：

> Strict checks are **enabled by default in Nuxt when the `typescript.typeCheck` option is enabled** to give you greater type safety.

**这句话有两层**：严格检查跟着 `typeCheck` 走；而 `typeCheck` 默认关。也就是说没开类型检查的项目，既没在做检查，也没有"strict 生效"这件事发生。

纯迁移期想先松一点，官方给的口子是：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    strict: false,
  },
})
```

## 收口：自动导入为什么"有类型"

150 那条承诺在这里落地成三件具体的东西：

| 机制 | 谁提供类型 |
| --- | --- |
| 自动导入名单 | 生成的类型声明（`prepare`/dev/build 时写入 `.nuxt`），tsconfig 里带 "references to auto-imports" |
| `#imports` 别名本身 | 同上——它是 path alias，写进生成的 tsconfig（154） |
| `kit` 的 `type` / `typeFrom` 字段 | 官方定义就是控制"生成类型声明时用什么来源"（"Use this as the `from` value when generating type declarations."） |
| 服务端那些 builder 特定类型 | `.nuxt/nuxt.d.ts`（缺了就退化成 builder-agnostic 类型，154 引过） |

**`#imports` 在服务端文件里要加 `@ts-expect-error`**（154 那句 "`#imports` is typed for the app, not the server build"）现在有了机制解释：app 与 server 是两份 tsconfig、两套声明，别名只属于前者。

## 装机与 CI 的落地清单（我的做法，非官方规定）

1. `package.json` 里加 `postinstall: nuxt prepare`——官方明确点名这个用法。
2. CI 里单独跑一步 `nuxt typecheck`，别指望 `build` 顺带查类型（默认不查）。
3. 记住 `prepare` 把 `NODE_ENV` 设成 `production`：若脚本依赖开发环境分支，别把它当 dev 用。
4. 类型增强只按目录放：`app/` / `server/` / `shared/`；改编译器选项只走 `typescript` 那五个键。
5. 新增一个自动导入目录（153）后，重跑一次 `prepare`，否则编辑器看不到新名字。

## 自查

1. 自动导入的类型存在哪个目录？三个生成时机分别是什么？
2. `nuxt prepare` 官方点名适合哪两个用途？它把 `process.env.NODE_ENV` 设成什么？
3. `--dotenv` 重复给多个路径时谁赢？`--envName` 在 build 与 dev server 下的默认值分别是什么？
4. 四个 tsconfig 各自管哪片代码？根 `tsconfig.json` 为什么是 `"files": []` 加四条 `references`？
5. `.nuxt/tsconfig.json` 还会生成吗？官方对它未来的说法是什么？
6. 类型增强为什么必须放对目录？官方那句"unless…"留了什么口子？
7. 想改 `lib`/`jsx`，该用 `tsConfig` 还是 `appTsConfig`？依据是哪句？
8. `nuxt dev`/`nuxt build` 默认会做类型检查吗？开检查要装什么、怎么开？strict 检查跟哪个选项联动？
9. 用一句话说清"为什么服务端文件里从 `#imports` 导入会报类型错"。

## 下一篇

156 转入组件的自动注册：默认只扫哪一个目录、目录数组的扫描顺序为什么是"嵌套的先加"，以及 `pattern` 一指定、`extensions` 就失效这条互斥。

## 本文依据

- `https://nuxt.com/docs/4.x/guide/concepts/typescript`
  - 原文（Type-checking）："By default, Nuxt doesn't check types when you run `nuxt dev` or `nuxt build`, for performance reasons."；"To enable type-checking at build or development time, install `vue-tsc` and `typescript` as development dependency:" + `npm install --save-dev vue-tsc typescript`（含 yarn/pnpm/bun/deno 四种写法）；"Then, run `nuxt typecheck` command to check your types:" + `npx nuxt typecheck`；"To enable type-checking at build or development time, you can also use the `typescript.typeCheck` option in your `nuxt.config` file:" + `typescript: { typeCheck: true }`
  - 原文（Auto-generated Types）："Nuxt projects rely on auto-generated types to work properly. These types are stored in the `.nuxt` directory and are generated when you run the dev server or build your application. You can also generate these files manually by running `nuxt prepare`."；"The generated `tsconfig.json` files inside the `.nuxt` directory include recommended basic TypeScript configuration for your project, references to auto-imports, API route types, path aliases like `#imports`, `~/file`, or `#build/file`, and more."；warning "Nuxt relies on this configuration, and Nuxt modules can extend it as well. For this reason, it is not recommended to modify your `tsconfig.json` file directly, as doing so could overwrite important settings. Instead, extend it via `nuxt.config.ts`."
  - 原文（Project References）："Nuxt uses TypeScript project references to improve type-checking performance and provide better IDE support. This feature allows TypeScript to break up your codebase into smaller, more manageable pieces."；"When you run `nuxt dev`, `nuxt build` or `nuxt prepare`, Nuxt will generate multiple `tsconfig.json` files for different parts of your application."；四个文件定义："`.nuxt/tsconfig.app.json` - Configuration for your application code within the `app/` directory"、"`.nuxt/tsconfig.node.json` - Configuration for your `nuxt.config.ts` and files outside the other contexts"、"`.nuxt/tsconfig.server.json` - Configuration for server-side code (when applicable)"、"`.nuxt/tsconfig.shared.json` - For code shared between app and server contexts (like types and non-environment specific utilities)"；"Each of these files is configured to reference the appropriate dependencies and provide optimal type-checking for their specific context."；note "For backward compatibility, Nuxt still generates `.nuxt/tsconfig.json`. However, we recommend using TypeScript project references with the new configuration files (`.nuxt/tsconfig.app.json`, `.nuxt/tsconfig.server.json`, etc.) for better type safety and performance. This legacy file will be removed in a future version of Nuxt."
  - 原文（Benefits 四条）："Faster builds: TypeScript can skip rebuilding unchanged projects"；"Better IDE performance: Your IDE can provide faster IntelliSense and error checking"；"Isolated compilation: Errors in one part of your application don't prevent compilation of other parts"；"Clearer dependency management: Each project explicitly declares its dependencies"
  - 原文（Augmenting Types with Project References）："Since the project is divided into multiple type contexts, it's important to augment types within the correct context to ensure they're properly recognized. TypeScript will not recognize augmentations placed outside these directories unless they are explicitly included in the appropriate context."；"For example, if you want to augment types for the `app` context, the augmentation file should be placed in the `app/` directory."；"For the `server` context, place the augmentation file in the `server/` directory."；"For types that are shared between the app and server, place the file in the `shared/` directory."；read-more 指向 `guide/modules/recipes-advanced#extend-typescript-config`
  - 原文（Strict Checks）："Strict checks are enabled by default in Nuxt when the `typescript.typeCheck` option is enabled to give you greater type safety."；"If you are currently converting your codebase to TypeScript, you may want to temporarily disable strict checks by setting `strict` to `false` in your `nuxt.config`:" + `typescript: { strict: false }`
- `https://nuxt.com/docs/4.x/api/commands/prepare`
  - 原文："The prepare command creates a .nuxt directory in your application and generates types."；"The `prepare` command creates a `.nuxt` directory in your application and generates types. This can be useful in a CI environment or as a `postinstall` command in your `package.json`."；命令行 `npx nuxt prepare [ROOTDIR] [--cwd=<directory>] [--dotenv=<path>...] [--logLevel=<silent|info|verbose>] [--envName=<environment>] [-e, --extends=<layer-name>...]`；参数表 `ROOTDIR` — "The root directory of your Nuxt project (default: `.`)"；选项表 `--cwd`、`--dotenv`（"Can be repeated, with later files taking precedence."）、`--logLevel`、`--envName`（"default is `production` when building, and `development` when running the dev server"）、`-e, --extends`（"Extend from a Nuxt layer"）；note "This command sets `process.env.NODE_ENV` to `production`."
- `https://nuxt.com/docs/4.x/directory-structure/tsconfig`
  - 原文："Nuxt automatically generates multiple TypeScript configuration files (`.nuxt/tsconfig.app.json`, `.nuxt/tsconfig.server.json`, `.nuxt/tsconfig.node.json` and `.nuxt/tsconfig.shared.json`) that include recommended basic TypeScript configuration for your project, references to auto-imports, API route types, path aliases, and more."；根 `tsconfig.json` 的 `{ "files": [], "references": [...] }` 四条 path 示例；warning "We do not recommend modifying the contents of this file directly, as doing so could overwrite important settings that Nuxt or other modules rely on. Instead, extend it via `nuxt.config.ts`."
  - 原文（Extending TypeScript Configuration）："You can customize the TypeScript configuration of your Nuxt project in the `nuxt.config.ts` file: set shared `compilerOptions` for every context at once with `typescript.tsConfig`, and override them for each context (`app`, `shared`, `node`, and `server`) individually."；示例键 `tsConfig`（注释 "shared compiler options for every generated tsconfig"）、`appTsConfig`、`sharedTsConfig`、`nodeTsConfig`、`serverTsConfig`；note "Most `compilerOptions` set in `typescript.tsConfig` are shared with every context, but a few are not. DOM- and Vue-specific options (such as `lib`, `jsx` and `jsxImportSource`) only make sense for your application code, so they are applied to `tsconfig.app.json` alone. Nuxt also manages `types`, `paths` and `noEmit` per context (the `node`, `shared` and `server` configs deliberately emit nothing and scan no ambient types), so setting these in `typescript.tsConfig` will not change them. Use the matching per-context option (`appTsConfig`, `nodeTsConfig`, `sharedTsConfig` or `serverTsConfig`) when you need to override them."；note "`typescript.serverTsConfig` extends `tsconfig.server.json`. `nitro.typescript.tsConfig` is kept in sync with it but is deprecated, so set `typescript.serverTsConfig` instead."
- `https://nuxt.com/docs/4.x/directory-structure/nuxt`
  - 对照用（见 087）：`.nuxt` 目录与 `buildDir`/`typesDir` 的分离
- `https://nuxt.com/docs/4.x/guide/modules/server-compatibility`
  - 对照用（见 154）："Builder-specific types... come from the generated `.nuxt/nuxt.d.ts`. Without it, they resolve to builder-agnostic types"；`// @ts-expect-error \`#imports\` is typed for the app, not the server build`
