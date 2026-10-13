---
url: /posts/3120ca3761f786999985d4b09236e1f6/
title: 491 - 搭测试环境：vitest.config 里的三个 project
date: 2027-03-24T11:00:00+08:00
updated: 2027-03-24T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方给的 `vitest.config.ts` 用 **三个 project** 分环境：`unit`（`include: ['test/unit/*.{test,spec}.ts']`、`environment: 'node'`）、`e2e`（`test/e2e/...`、同样是 `node`）、`nuxt`（`test/nuxt/...`、`environment: 'nuxt'`，且只有这一个包在 `defineVitestProject` 里）。那条 `::note` 是新手最容易配错的地方：**`defineVitestProject` 只用于 Nuxt 环境的测试，端到端应当按普通 `test.environment: 'node'` 的 project 配**。Nuxt 环境下跑测试时有两条硬后果：它跑在 `happy-dom` 或 `jsdom` 里，且**测试开始前会初始化一个全局 Nuxt 应用（包括你在 `app.vue` 里写的代码与插件）**，所以官方警告不要改全局状态、非改不可就要复位。跑单套用 `npx vitest --project unit` / `--project nuxt`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - vitest
  - environment
  - 全局状态
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`vitest.config.ts` 里把 `environment` 全设成 `'nuxt'`，跑起来慢得离谱，而且几个纯函数的测试开始报"找不到 Nuxt 实例"这种莫名其妙的错。

官方那份配置的形状是三个 project，各配各的环境。这一篇逐行拆，包括那句关于全局状态的警告。

## 官方那份配置

Setup 第二步给的完整内容：

```ts twoslash
import { defineConfig } from 'vitest/config'
import { defineVitestProject } from '@nuxt/test-utils/config'

export default defineConfig({
  test: {
    projects: [
      {
        test: {
          name: 'unit',
          include: ['test/unit/*.{test,spec}.ts'],
          environment: 'node',
        },
      },
      {
        test: {
          name: 'e2e',
          include: ['test/e2e/*.{test,spec}.ts'],
          environment: 'node',
        },
      },
      await defineVitestProject({
        test: {
          name: 'nuxt',
          include: ['test/nuxt/*.{test,spec}.ts'],
          environment: 'nuxt',
        },
      }),
    ],
  },
})
```

读出来的四件事：

| project | include | environment | 包装方式 |
| --- | --- | --- | --- |
| `unit` | `test/unit/*.{test,spec}.ts` | `node` | 普通对象 |
| `e2e` | `test/e2e/*.{test,spec}.ts` | `node` | 普通对象 |
| `nuxt` | `test/nuxt/*.{test,spec}.ts` | `nuxt` | **`await defineVitestProject({...})`** |

**三个 `name` 就是命令行上 `--project` 的取值**（见本节末）。

紧跟着一条 `::note` 是这一篇的关键纠错点：

> `defineVitestProject` is only for Nuxt-environment tests. End-to-end tests should be configured as a regular `test.environment: 'node'` project.

也就是：**别把 e2e 塞进 `defineVitestProject`**，它就是普通 node project。上面那张表里 `e2e` 与 `unit` 用的是普通对象，正是这条 note 的落地形状。

## 为什么要分：官方给的理由

Using a Nuxt Runtime Environment 一节说清了每个环境的用途：

> Using Vitest projects, you have fine-grained control over which tests run in which environment:
>
> - **Unit tests**: Place regular unit tests in `test/unit/` - these run in a Node environment for speed
> - **Nuxt tests**: Place tests that rely on the Nuxt runtime environment in `test/nuxt/` - these will run within a Nuxt runtime environment

分开的收益官方用了两个字：**for speed**。纯逻辑测试不必起 Nuxt 应用。

## 简单配法与它被警告的原因

官方也给了"全部跑在 Nuxt 环境"的省事的配法：

```ts twoslash
import { defineVitestConfig } from '@nuxt/test-utils/config'
import { fileURLToPath } from 'node:url'

export default defineVitestConfig({
  test: {
    environment: 'nuxt',
    // you can optionally set Nuxt-specific environment options
    // environmentOptions: {
    //   nuxt: {
    //     rootDir: fileURLToPath(new URL('./playground', import.meta.url)),
    //     domEnvironment: 'happy-dom', // 'happy-dom' (default) or 'jsdom'
    //     overrides: {
    //       // other Nuxt config you want to pass
    //     }
    //   }
    // }
  },
})
```

注释里那三项是 Nuxt 环境自己的选项：**`rootDir`**（指到另一个 Nuxt 项目，例如 playground）、**`domEnvironment`（默认 `happy-dom`，另一个取值是 `jsdom`）**、**`overrides`**（额外要传进去的 Nuxt 配置）。

用这种默认全 Nuxt 的配法时，可以按文件退出环境：

```ts twoslash
// @vitest-environment node
import { test } from 'vitest'

test('my test', () => {
  // ... test without Nuxt environment!
})
```

官方对这条路的态度是一条 `::warning`，理由写得很技术：

> This approach is not recommended as it creates a hybrid environment where Nuxt Vite plugins run but the Nuxt entry and `nuxtApp` are not initialized. This can lead to hard-to-debug errors.

**这是一种混合环境：Nuxt 的 Vite 插件在跑，但 Nuxt 入口与 `nuxtApp` 没被初始化**，后果是难查的报错。所以"整个仓库都设成 nuxt 环境、个别文件再退回 node"这条路官方不推荐——**回到三个 project 那份配置**。

## 跑起来与那两条环境事实

官方给的四种跑法：

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

然后是这一篇最该记住的一条。`::warning` 原文：

> When you run your tests within the Nuxt environment, they will be running in a `happy-dom` or `jsdom` environment. Before your tests run, a global Nuxt app will be initialized (including, for example, running any plugins or code you've defined in your `app.vue`).
>
> This means you should take particular care not to mutate the global state in your tests (or, if you need to, to reset it afterwards).

三件事连在一起：**DOM 层是 happy-dom 或 jsdom**、**测试跑之前会初始化一个全局 Nuxt 应用**、**这个初始化会执行你在 `app.vue` 里写的代码和插件**。

推论就是官方那句警告：**别在测试里改全局状态；必须改就事后复位**。同一次进程里多个测试文件共享的正是这个全局应用——一个文件留下的脏状态会喂给下一个文件，这类"单跑通过、合跑失败"的现象从这里来。

## 目录放错地方的补救

Setup 第三步与后面那节对接：**Nuxt 环境的测试若不在 `test/nuxt/`**，类型上下文要手动补：

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

注释那句限重要：**这个路径是相对于生成的 `.nuxt/tsconfig.json` 的**（所以出现 `../`）。默认被纳入的只有 `test/nuxt/` 与 `tests/nuxt/` 两个目录——**顶层的 `nuxt` 那一段是条件**，别的嵌套深度不自动算。

但官方紧接着用一条 `::important` 划了使用边界：

> Unit tests should not depend on Nuxt runtime features like auto-imports or composables. Only add TypeScript path alias support if your tests import from your source files (e.g., `~/utils/helpers`), not for Nuxt-specific features.

**加路径别名的唯一正当理由是"测试真的从源码文件导入"，而不是为了让单元测试能用自动导入或组合式函数。** 这一句也是 490 那句"单元测试该跑在 node 里"的延伸：不该依赖运行时特性。

## 自查

- 三个 project：`unit` 与 `e2e` 都是 `environment: 'node'` 的普通对象，只有 `nuxt` 用 `await defineVitestProject` 且 `environment: 'nuxt'`。
- `defineVitestProject` 只给 Nuxt 环境测试用，e2e 别塞进去。
- 分开配的理由是速度：纯逻辑测试不必起 Nuxt 应用。
- 全设 nuxt、个别文件用 `// @vitest-environment node` 退出是官方**不推荐**的混合环境（Vite 插件跑、`nuxtApp` 未初始化）。
- Nuxt 环境跑测试前会初始化全局 Nuxt 应用并执行 `app.vue` 与插件代码，所以别改全局状态、改了就复位。
- 补类型目录走 `typescript.tsConfig.include`，路径相对 `.nuxt/tsconfig.json`；但别为了让单元测试用自动导入而加它。

## 下一篇

下一篇 492 收这套工具的能力边界：哪些是 `@nuxt/test-utils` 自己的模块（`/config`、`/e2e`、`/nuxt` 之外的路径、以及那个可选的 `/module`）、mock 有哪三样、helpers 与 `mountSuspended` 之类工具各自出现在哪一节，以及官方对"其他 runner"和"其他 runtime"分别是什么态度。

## 本文依据

- 测试页 Unit Testing 一节：Setup 第二步的完整 `vitest.config.ts`（三个 project 的 `name`/`include`/`environment` 与 `await defineVitestProject`）、`::note`（defineVitestProject 只用于 Nuxt 环境、e2e 应为普通 `test.environment: 'node'` project）、Setup 第三步与 Adding other test directories（`typescript.tsConfig.include` 及其"路径相对生成的 .nuxt/tsconfig.json"注释、默认只纳入 `test/nuxt/` 与 `tests/nuxt/`）、`::important`（单元测试不该依赖自动导入与组合式函数、只在真的从源码导入时加别名）（`getting-started/testing`）
- Using a Nuxt Runtime Environment 两条（unit 在 `test/unit/` 跑 Node 为速度、Nuxt 测试在 `test/nuxt/`）；Alternative Simple Setup 的 `defineVitestConfig` 示例与注释里的 `rootDir`/`domEnvironment`（默认 happy-dom）/`overrides`；按文件退出的 `// @vitest-environment node` 示例与其 `::warning`（hybrid environment、Nuxt Vite plugins run but Nuxt entry and `nuxtApp` not initialized、hard-to-debug errors）；Running Tests 四条命令；`::warning`（happy-dom 或 jsdom、跑前初始化全局 Nuxt 应用含 `app.vue` 与插件、不要改全局状态或事后复位）（`getting-started/testing`）
- `test/nuxt/` 与 `tests/nuxt/` 的类型上下文与 `typescript.tsConfig.include` 的关系（`guide/concepts/typescript`、069/395 依据）
