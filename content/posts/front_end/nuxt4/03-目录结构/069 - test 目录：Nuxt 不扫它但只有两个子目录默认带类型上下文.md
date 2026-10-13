---
url: /posts/09b5d42fd34d4a3552e36bbbb6f780e9/
title: 069 - test 目录：Nuxt 不扫它，但只有两个子目录默认带类型上下文
date: 2026-11-04T09:00:00+08:00
updated: 2026-11-04T09:00:00+08:00
author: cmdragon

summary:
  摘要：test/ 目录是官方推荐放置应用测试（单元、Nuxt 运行时与端到端）的位置，但它与其他目录有一个根本区别——官方明确"Nuxt 不会像扫描 app/ 或 server/ 那样扫描它"，你需要自己选运行器与布局（通常配 @nuxt/test-utils）。官方给的常见布局按环境分成三块并各自定义：test/unit/ 是不带 Nuxt 运行时的快速 Node 测试、test/nuxt/ 是需要 Nuxt 运行时环境的测试、test/e2e/ 是针对运行中应用做的端到端测试；官方同时说明你当然可以选任何结构，但把需要 Nuxt 运行时环境的测试与端到端测试分开对测试稳定性很重要。类型方面只有一条默认规则值得记住：test/nuxt/ 与 tests/nuxt/ 里的测试文件默认被包含进 Nuxt 应用类型上下文，因此它们能识别 ~~/、@/、#imports 这些 Nuxt 别名并感知在应用里可用的自动导入，而 test/unit/ 这类目录要手动加；手动添加的入口是 typescript.tsConfig.include，且官方在示例注释里点明"这个路径是相对于生成的 .nuxt/tsconfig.json 的"。另外官方用 important 写着：单元测试不应依赖自动导入或组合式函数这类 Nuxt 运行时特性。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - vitest
  - 类型上下文
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前面每一篇目录文章都有一个共同前提：**文件放进对的目录，框架就自动接手。** `test/` 是第一个打破这个前提的目录——官方一上来就告诉你它不扫。

## 第一句就与别的目录不同

> `test/` 目录是你应用测试的**推荐**位置。**Nuxt 不会像扫描 `app/` 或 `server/` 那样扫描它；你自己选择运行器与布局**（通常配合 `@nuxt/test-utils`）。

**这句话推翻了一个很自然的推想：不要以为建了 `test/` 就有什么会被自动注册或自动跑起来。**

对照一下这一串目录：

| 目录 | 框架参与程度 |
| --- | --- |
| `app/components/`、`app/middleware/`、`server/api/`… | 扫描 + 自动注册（050 那张表） |
| `shared/` | 扫描两个子目录并自动导入（066） |
| **`test/`** | **不扫描；你选运行器与布局** |

**而"推荐"这个词官方用得也准：它是建议，不是要求。**

## 那份布局：按环境切三块

> 一个常见的布局是按环境分隔：

```bash [Directory structure]
-| test/
---| e2e/
---| nuxt/
---| unit/
```

> - `test/unit/` —— **不带 Nuxt 运行时的快速 Node 测试**
> - `test/nuxt/` —— **需要 Nuxt 运行时环境的测试**
> - `test/e2e/` —— **针对运行中的应用做的端到端测试**

Getting Started 那一页给了带文件的版本：

```bash [Directory structure]
test/
├── e2e/
│   └── ssr.test.ts
├── nuxt/
│   ├── components.test.ts
│   └── composables.test.ts
├── unit/
│   └── utils.test.ts
```

> 你当然可以选择任何测试结构，**但把 Nuxt 运行时环境的测试与 Nuxt 端到端测试分开，对测试稳定性很重要。**

**这句官方理由是"稳定性"，不是"可读性"。** 为什么分开才稳？下面那条 warning 给了原因——**两种环境跑的东西根本不是一套东西。**

## 只认两个目录名：`test/nuxt` 与 `tests/nuxt`

类型这一节是这一篇最实用的部分：

> 默认情况下，**`test/nuxt/` 与 `tests/nuxt/` 目录里的测试文件被包含进 Nuxt 应用类型上下文。** 这意味着**它们能识别 Nuxt 别名（像 `~/`、`@/`、`#imports`），而 TypeScript 也会知道那些在你的 Nuxt 应用里可用的自动导入。**

> （tip）这与推荐结构一致——**只有需要 Nuxt 运行时环境的测试才被放进这些目录。像 `test/unit/` 这种其它目录里的单元测试，如有需要可以手动添加。**

**读出三件事：**

1. **默认带应用类型上下文的只有这两个目录名**（`test/nuxt/` 与 `tests/nuxt/`，注意**复数那个也算**）。
2. **`test/unit/` 默认不在应用上下文里**——**这正是它应有的样子**，因为按官方定义它"不带 Nuxt 运行时"。
3. 031、036 那套"四份 tsconfig 按上下文分"在这里继续生效：**应用上下文 = `tsconfig.app.json` 那一族。**

## 想加别的目录：`include` 那条，以及基准路径

> 如果你在其它目录里也有测试、并且你打算在 Nuxt Vitest 环境里运行它们，你可以通过把它们加进配置，**把它们包含进 Nuxt 应用类型上下文**：

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

**注意那行注释，它是这一节最值钱的一句：**

> this path is relative to the generated **`.nuxt/tsconfig.json`**

**也就是说这个路径的基准不是项目根，而是生成目录里那份配置——所以才写成 `../test/...`。** 051 讲过 `srcDir`/`serverDir` 的基准是 `rootDir`，016 附近那几篇讲过 `.nuxt` 是生成物；**这里是第三种类基准。**（036 讲过：`typescript.tsConfig` 的多数选项共享，但 `include` 这类写在生成配置里的路径要按它的实际基准来算。）

## 官方那条 important：单元测试别依赖运行时特性

> **单元测试不应依赖像自动导入或组合式函数这样的 Nuxt 运行时特性。**
>
> **只有当你的测试从你的源文件里导入东西（例如 `~/utils/helpers`）时，才添加 TypeScript 路径别名支持——不要为了 Nuxt 专属特性而添加。**

**这第二句实际上把"什么时候该去配 `include`"给判死了：**

- ✅ 你的单元/其它目录测试里写了 `import { helpers } from '~/utils/helpers'` → 加 `include` 让它认得别名，合理。
- ❌ 你只是想在单元测试里直接用自动导入的 `useFoo()`（063）→ **官方的立场是：单元测试不该依赖这些。**

**这条与 063 那句"这个目录不提供额外响应式能力"是同一种态度：自动导入是框架在应用上下文里给的便利，不是语言特性。**

## 运行：Vitest 的 project 名来自你的配置

> 有了 project 设置，你可以运行不同的测试套件：

```bash
# Run all tests
npx vitest

# Run only unit tests
npx vitest --project unit

# Run only Nuxt tests
npx vitest --project nuxt

# Run tests in watch mode
npx vitest --watch
```

**注意 `--project unit` / `--project nuxt` 这两个值，与前面那份目录布局（`test/unit/`、`test/nuxt/`）同名**——这是官方示例里的对应关系（Vitest 的 project 配置本身在 490-509 的 491 那节展开）。

## 为什么必须把两类测试分开：那条 warning

> 当你在 Nuxt 环境里运行测试时，**它们会运行在一个 `happy-dom` 或 `jsdom` 环境里。在你的测试开始之前，一个全局的 Nuxt 应用会被初始化（包括例如运行你在 `app.vue` 里定义的代码、以及任何插件）。**
>
> **这意味着你应当特别小心不要改动全局状态（或者，如果你需要改，之后把它重置）。**

**这段同时给出了三个因果，都落在前面读过的目录上：**

1. **Nuxt 环境的测试会跑你的插件**（061 那整页的 `app/plugins/`）——**插件在测试启动前就执行了。**
2. **会跑 `app.vue` 里的代码**（054 那条 note：加进 `app.vue` 的 JS 是全局的、每个页面都执行）——**在测试里同样执行。**
3. **DOM 环境是 `happy-dom` 或 `jsdom`**，不是真浏览器。

**而端到端测试（`test/e2e/`）打的是"运行中的应用"**（069 开头那句定义）——**它跑的是真实构建/真实服务器，不是这个被初始化出来的全局 Nuxt 实例。** 这就是"分开对稳定性很重要"的具体含义。

**"不要改全局状态"这句我建议在 `test/nuxt/` 里当硬规矩执行**：一个测试改了 `useState` 或 `useAppConfig`（055）留下的值，下一个测试会读到脏状态（这条后果是我按"全局 Nuxt 应用 + 不重置"这两个官方事实推的）。

## 自查

- [ ] `test/` 与其他目录最本质的区别是什么？官方怎么说？
- [ ] 三个子目录各自对应"要不要 Nuxt 运行时"的哪一档？
- [ ] 默认被放进 Nuxt 应用类型上下文的是哪两个目录名？
- [ ] 往 `typescript.tsConfig.include` 里加测试目录时，路径基准是什么？
- [ ] 官方允许单元测试依赖自动导入吗？那条 important 的原文判断标准是什么？
- [ ] 在 Nuxt 环境里跑测试之前发生了什么？因此要注意什么？

## 下一篇

070 讲 `content/`：这个目录由 Nuxt Content 模块启用而不是框架自带，官方给的可读文件类型与"靠 catch-all 页面 + `queryCollection` 按 `route.path` 取内容"的实现方式。

## 本文依据

- `directory-structure/test`（标题：使用 test/ 目录组织你应用的单元、Nuxt 与端到端测试；test/ 是应用测试的推荐位置、Nuxt 不会像扫描 app/ 或 server/ 那样扫描它、你自己选择运行器与布局（通常配合 @nuxt/test-utils）；常见布局按环境分隔为 e2e/nuxt/unit 三块，并给出 test/unit/ 是不带 Nuxt 运行时的快速 Node 测试、test/nuxt/ 是需要 Nuxt 运行时环境的测试、test/e2e/ 是针对运行中应用的端到端测试；read-more 指向 getting-started/testing#organizing-your-tests 并说明那节含 setup、Vitest projects 与 TypeScript context）
- `getting-started/testing`（Organizing Your Tests：project-based setup 下给出的目录树示例 test/e2e/ssr.test.ts、test/nuxt/components.test.ts 与 composables.test.ts、test/unit/utils.test.ts，并说明你当然可以选择任何测试结构、但把需要 Nuxt 运行时环境的测试与 Nuxt 端到端测试分开对测试稳定性很重要；TypeScript Support in Tests：默认 test/nuxt/ 与 tests/nuxt/ 里的测试文件被包含进 Nuxt 应用类型上下文，因此它们识别 Nuxt 别名（~/、@/、#imports）且 TypeScript 知道在应用里可用的自动导入，tip 这与推荐结构一致即只有需要 Nuxt 运行时环境的测试放在这些目录而 test/unit/ 这类目录如需可手动加入；Adding other test directories：若其它目录的测试将在 Nuxt Vitest 环境运行可通过 typescript.tsConfig.include 加入 Nuxt 应用类型上下文，示例 include 里注释 this path is relative to the generated .nuxt/tsconfig.json 并使用 '../test/other-nuxt-context/**/*'；important 单元测试不应依赖自动导入或组合式函数这类 Nuxt 运行时特性，只有当测试从源文件导入（如 ~/utils/helpers）时才添加 TypeScript 路径别名支持而不是为了 Nuxt 专属特性；Running Tests：npx vitest 跑全部、npx vitest --project unit 只跑单元、--project nuxt 只跑 Nuxt 测试、--watch 监听模式；warning 在 Nuxt 环境里运行测试会运行在 happy-dom 或 jsdom 环境、测试开始前会初始化一个全局 Nuxt 应用（包括运行插件与你在 app.vue 里定义的代码），因此应特别小心不要修改全局状态（或需要时事后重置））
- `directory-structure/app/plugins`（对照用：插件在 Nuxt 环境测试初始化时会被执行；官方也建议优先用 composables 而非全局 provide）
- `directory-structure/app/app`（对照用：加进 app.vue 的 JS 与 CSS 是全局的并被包含进每一个页面）
- `directory-structure/app/composables`（对照用：自动导入是应用上下文的便利，单元测试不应依赖）
