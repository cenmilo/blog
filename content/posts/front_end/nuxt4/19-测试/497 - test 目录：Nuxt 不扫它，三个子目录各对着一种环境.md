---
url: /posts/9040f7732908581d7a84d967e13335b4/
title: test 目录：Nuxt 不扫它，三个子目录各对着一种环境
date: 2027-03-26T11:00:00+08:00
updated: 2027-03-26T11:00:00+08:00
author: cmdragon

summary:
  摘要：`test/` 是官方推荐的放测试文件的目录，但它与 `app/`、`server/` 有本质区别：**Nuxt 不会扫描它**。原文是 "Nuxt does not scan it the way it does `app/` or `server/`; you choose the runner and layout yourself"。所以往这个目录扔文件不会自动发生任何事，文件名也只有一处官方约定——`.nuxt.spec.ts` 用来标记跑在 Nuxt 运行时的测试文件。这一页给的推荐布局是三个子目录：`test/unit/`（不需要 Nuxt 运行环境的快速 Node 测试）、`test/nuxt/`（需要 Nuxt 运行环境的测试）、`test/e2e/`（对着跑起来的应用做端到端测试）。测试文件名后缀在官方示例里本身就不统一，这篇一并列清楚。

categories:
  - 前端开发

tags:
  - nuxt4
  - 目录结构
  - 测试
  - vitest
  - 约定
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

你在 `app/components/` 里新建一个 `.vue`，模板里直接就能用，不用 import。到了 `test/` 目录，你以为放个 `xxx.test.ts` 就会被跑起来——结果一条测试也没执行。

因为这两个目录的机制完全不同。

## 这一页最关键的一句

`directory-structure/test` 这一页正文不长，定性全在第一段：

> The `test/` directory is the recommended place for your application tests. Nuxt does not scan it the way it does `app/` or `server/`; you choose the runner and layout yourself (typically with `@nuxt/test-utils`).

三个信息点，逐个落到你身上：

**它是"推荐"，不是"约定"。** 与 `app/pages/` 那种"放进去了就生效"不同，这个目录名不触发任何框架行为。

**Nuxt 不扫它。** 也就是：没有自动注册、没有按目录生成路由、没有因为文件名而改变行为。你写的每个测试都要由测试运行器自己去发现，而发现规则写在你的 Vitest 配置里，跟 Nuxt 无关。

**运行器和布局你自己选。** 括号里那句 "typically with `@nuxt/test-utils`" 是常见做法，不是强制。

这一句也解释了一个常见现象：`app/` 里的文件放错目录你会立刻看到报错，`test/` 里放错了你只会看到"测试通过"——因为一条都没跑。判断办法是看运行器的输出里被执行的文件数，而不是看目录里有没有文件。

## 推荐的三个子目录

页面给的目录树就三层：

```bash [Directory structure]
-| test/
---| e2e/
---| nuxt/
---| unit/
```

对应的三条定义，逐字按文档：

| 子目录 | 官方定义 | 需不需要 Nuxt 运行环境 | 需不需要一个跑着的应用 |
| --- | --- | --- | --- |
| `test/unit/` | fast Node tests without the Nuxt runtime | 不需要 | 不需要 |
| `test/nuxt/` | tests that need the Nuxt runtime environment | 需要 | 不需要 |
| `test/e2e/` | end-to-end tests against a running app | 需要（是真实应用） | 需要 |

分界线只有一条：**这段代码离开 Nuxt 环境还能不能跑**。纯函数、工具函数放 `unit/`；要用自动导入、要用 `useState` 这类组合式函数、要挂组件的放 `nuxt/`；要看服务器渲染出来的 HTML 或者要操作浏览器的放 `e2e/`。

页面末尾留了一个跳转，指向测试文档里的 **Organizing Your Tests** 一节——那节讲怎么按这三类配 Vitest project、TypeScript 上下文怎么带上测试目录。这一页只负责说目录放哪，配置不在这页。

## 文件后缀：一处真约定，其余是混用

`directory-structure/test` 这一页确实没给文件名规则。但**测试页给了一条**——在讲 `/runtime` 与 `/e2e` 不能同文件那一节，官方把后缀当成选环境的手段：与其用 `// @vitest-environment nuxt` 注释，"or name your runtime unit test files with the `.nuxt.spec.ts` extension"，并给出配对的 `app.e2e.spec.ts`。所以后缀不是纯装饰，`.nuxt.spec.ts` 被官方用来标记"这个文件要跑在 Nuxt 运行时里"。

除此之外能看到的只有其他页的示例用了哪些名字：

| 示例文件 | 出现的页面 | 所在目录 |
| --- | --- | --- |
| `test/my-test.spec.ts` | `getting-started/testing`（e2e Setup） | `test/` |
| `app.nuxt.spec.ts` | `getting-started/testing`（e2e 冲突那节的运行侧示例） | 未标目录 |
| `app.e2e.spec.ts` | 同上（端到端侧示例，内含 `setup({ setupTimeout: 10000 })`） | 未标目录 |
| `test/e2e/ssr.test.ts` | `getting-started/testing`（Organizing Your Tests） | `test/e2e/` |
| `test/nuxt/components.test.ts` | 同上 | `test/nuxt/` |
| `test/nuxt/composables.test.ts` | 同上 | `test/nuxt/` |
| `test/unit/utils.test.ts` | 同上 | `test/unit/` |
| `tests/components/SomeComponents.nuxt.spec.ts` | `getting-started/testing`（`mountSuspended`／`renderSuspended`） | `tests/` |
| `tests/App.nuxt.spec.ts` | 同上 | `tests/` |
| `test/nuxt/import.test.ts` | `getting-started/testing`（`mockNuxtImport` 带类型那条） | `test/nuxt/` |
| `test/fixtures/ssr/nuxt.config.ts` | `guide/modules/testing`（模块作者的 fixture） | `test/fixtures/` |
| `test/rendering.ts` | `guide/modules/testing` | `test/` |
| `tests/example.test.ts` | `getting-started/testing`（Playwright 运行器） | `tests/` |

从这张表能读出来的事实：

- **`test/` 与 `tests/` 两种写法在官方示例里都出现**，不是笔误级别的统一。测试文档里 TypeScript 那句直接写成 "test files in `test/nuxt/` and `tests/nuxt/` directories"，两个都算。
- **`.test.ts` 与 `.spec.ts` 都在用**，e2e 那个示例甚至同一页里两种都有。
- **`.nuxt.spec.ts` 是官方明说过的一条约定**（用它标记跑在 Nuxt 运行时里的单元测试文件，与 `// @vitest-environment nuxt` 注释二选一）；成对出现的 `.e2e.spec.ts` 只在那一节的示例文件名里出现，官方没有为它写规则句。

除 `.nuxt.spec.ts` 这一处，文件名不决定别的：哪个 project 抓到你的文件，看的是 Vitest 配置里的 `include`/`exclude`；而这个文件能不能用自动导入，看的是它落在哪个目录或用了哪种环境标记。

## 自查

- `test/` 是推荐位置，**Nuxt 不扫描它**：放文件不会自动生效，跑没跑要看运行器输出的文件数。
- 三个子目录按「离开 Nuxt 运行环境还能不能跑」分：`unit/` 不要环境、`nuxt/` 要环境、`e2e/` 要跑着的应用。
- 目录布局可以自选，官方那句提醒是把 Nuxt 运行环境的测试与端到端测试分开放，理由是测试稳定性。
- 唯一的后缀约定是 `.nuxt.spec.ts`（官方用它标记跑在 Nuxt 运行时的单元测试文件，与 `// @vitest-environment nuxt` 二选一）；`test/` 与 `tests/`、`.test.ts` 与 `.spec.ts` 在示例里仍然混用。
- 模块作者的测试有第四种目录：`test/fixtures/*` 放被测用的 Nuxt 应用。

## 下一篇

下一篇 498 讲怎么给自动导入的组合式函数写测试：`mockNuxtImport` 的三种写法、它为什么一个文件里只能对同一个导入用一次，以及 `vi.hoisted` 那条绕法。

## 本文依据

- https://nuxt.com/docs/4.x/directory-structure/test（整页：recommended place 与 "Nuxt does not scan it the way it does `app/` or `server/`" 的原句、目录树、三条子目录定义、指向 Organizing Your Tests 的 read-more）
- https://nuxt.com/docs/4.x/getting-started/testing（Organizing Your Tests 的目录树与四条文件；TypeScript 小节中 `test/nuxt/` 与 `tests/nuxt/` 并列；`mountSuspended`/`renderSuspended`/`mockNuxtImport`/Playwright 各示例的文件名；e2e Setup 示例 `test/my-test.spec.ts`）
- https://nuxt.com/docs/4.x/getting-started/testing（Conflict with End-To-End Testing 一节：两种环境不能同文件的原因句、`// @vitest-environment nuxt` 注释与 "name your runtime unit test files with the `.nuxt.spec.ts` extension"、`app.nuxt.spec.ts` 与 `app.e2e.spec.ts` 两份示例文件名）
- https://nuxt.com/docs/4.x/guide/modules/testing（`test/fixtures/ssr/nuxt.config.ts` 与 `test/rendering.ts`）
