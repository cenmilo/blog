---
url: /posts/46127ff9a70346124b1623216d387479/
title: 504 - 组织测试文件：官方 Organizing Your Tests 一节的四件事
date: 2027-03-29T09:00:00+08:00
updated: 2027-03-29T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方 Organizing Your Tests 那一节只讲四件事：一棵推荐目录树（`test/e2e/`、`test/nuxt/`、`test/unit/` 加四个示例文件）、一句可以自选结构但**必须把 Nuxt 运行环境与端到端测试分开的理由**（测试稳定性）、四条运行命令（全量、`--project unit`、`--project nuxt`、`--watch`）、以及 TypeScript 上下文的接法（默认只有 `test/nuxt/` 与 `tests/nuxt/` 在里面，别的目录要往 `typescript.tsConfig.include` 加，且路径相对生成的 `.nuxt/tsconfig.json`）。这一节没有任何文件名规则、没有测试分层理论，也没有讲并发。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - vitest
  - 目录结构
  - typescript
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

测试文件攒到十几个以后，你会遇到一个不像技术问题的问题：这个文件该放哪。

官方在 `getting-started/testing` 里确实有一节回答它，叫 Organizing Your Tests。这一节很短，能拿走的只有四件东西，而且第一件之前就有人把它读成了规范。

## 一、那棵目录树长什么样

原文的前提句是 "With the project-based setup, you might organize your tests as follows"——注意用词是 **might**，不是 must。给出的树带具体文件名：

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

四个示例文件的位置各对应一类被测对象：`ssr.test.ts` 在 `e2e/`、组件与组合式函数在 `nuxt/`、工具函数在 `unit/`。这份树与 `directory-structure/test` 那一节给的三行目录是同一套，只是那一节只列目录、这份树把文件也列了出来。

## 二、唯一一句硬性理由

紧接着那句才是这一节的核心：

> You can of course opt for any test structure, but keeping the Nuxt runtime environment separated from Nuxt end-to-end tests is important for test stability.

拆成两半：**结构随便你选**，但**运行环境类的测试与端到端类测试要分开**，理由是**测试稳定性**。

为什么分开会稳？同页另一处给了可以直接对上的两条事实。一条是环境差异：在 Nuxt 环境里跑的测试用的是 happy-dom 或 jsdom，测试执行前会初始化一个**全局 Nuxt 应用**（包括插件与 `app.vue` 里的代码），官方因此提醒不要改动全局状态；端到端那侧跑的是构建出来的应用，由 `setup()` 在 `beforeAll`/`beforeEach`/`afterEach`/`afterAll` 四个钩子里搭环境。另一条是导入冲突：`@nuxt/test-utils/runtime` 与 `@nuxt/test-utils/e2e` **需要跑在不同测试环境，因此不能出现在同一个文件里**。

也就是说，"混在一起"的具体后果是：全局应用被端到端的那套钩子反复搭拆，或者一个文件里同时请求两种环境。混着的代价不是不优雅，是同一批测试在不同执行顺序下给出不同结果。

## 三、四条运行命令

按 project 分开之后，官方的运行方式就是这四条，一字不多：

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

三个读数：

- 只有 `unit` 与 `nuxt` 两个 project 名出现在命令里，**`e2e` 没有对应的示例命令**。要按 project 跑端到端，写法是同一形式（`--project <name>`），但文档没给例子，具体名字以你自己配置里写的为准。
- `--watch` 是运行器参数，不是 project。
- 这一节的命令全是 `npx vitest` 开头的，**没有一条 `nuxt test`**。那条 CLI 命令在另一页（`api/commands/test`），只说"用 `@nuxt/test-utils` 跑测试"，并且没有按测试种类筛的选项。

## 四、TypeScript 上下文：默认范围与手工扩法

这一节里最容易被跳读、但最容易让人卡住的是这段。默认事实一句：

> By default, test files in `test/nuxt/` and `tests/nuxt/` directories are included in the Nuxt app TypeScript context.

在这个上下文里，测试文件才认 `~/`、`@/`、`#imports` 这类别名，TypeScript 才知道你的应用里那些自动导入。两个目录名（单数 `test/` 与复数 `tests/`）都被官方写进来了。

`::tip` 补的是选址含义：这与推荐结构一致——只有需要 Nuxt 运行环境的测试放这些目录，其它目录（比如 `test/unit/`）里的单元测试需要时手工加。

手工加的写法在 "Adding other test directories" 这个子小节：

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

两处细节值得单独记。第一，注释那行是**路径基准**：它相对的是生成的 `.nuxt/tsconfig.json`，所以示例里会出现 `../`。基准想当然写成项目根，表现就是加了 include 类型还是认不出来。第二，这条通道是给"在 Nuxt 环境里跑、但放在别的目录"的测试用的——浏览器模式的测试就在 `test/browser/`，官方在那里直接指回这个小节让你加。

然后 `::important` 把口子收住：

> Unit tests should not depend on Nuxt runtime features like auto-imports or composables. Only add TypeScript path alias support if your tests import from your source files (e.g., `~/utils/helpers`), not for Nuxt-specific features.

顺序因此很清楚：先看这段代码该不该用 Nuxt 运行时，再决定它放哪个目录、要不要进类型上下文。反过来做（为了让单元测试能用 `useState` 而加 include）就是官方明确不鼓励的那种。

## 自查

- 官方给的是**推荐**结构（"you might organize"），三目录四文件那棵树；结构可自选，但按环境分开有明确理由：测试稳定性。
- 分开的两条硬依据：Nuxt 环境会先初始化全局应用（别改全局状态），以及 `/runtime` 与 `/e2e` 不能同文件。
- 运行命令四条，`--project` 只举了 `unit` 与 `nuxt`，`e2e` 无示例；这一节没有 `nuxt test`。
- 类型上下文默认覆盖 `test/nuxt/` 与 `tests/nuxt/`；其它目录走 `typescript.tsConfig.include`，**路径相对生成的 `.nuxt/tsconfig.json`**。
- 加 include 的目的只能是源码路径导入，不是给单元测试开 Nuxt 特性的口子。

## 下一篇

下一篇 505 讲 DOM 侧的内置 mock：整页只列了两个（`intersectionObserver` 与 `indexedDB`），默认值一个是真一个是假，而且配置键的写法与标题不一样。
