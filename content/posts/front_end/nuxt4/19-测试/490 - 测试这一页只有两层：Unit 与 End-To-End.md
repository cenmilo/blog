---
url: /posts/11925e279cfbfb3983628a54ceb60227/
title: 490 - 测试这一页只有两层：Unit 与 End-To-End
date: 2027-03-24T10:00:00+08:00
updated: 2027-03-24T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方测试这一节的结构是**三节**：Installation、Unit Testing、End-To-End Testing——没有"集成测试"这一层，"组件测试"与"组合式函数测试"在 4.x 文档里只是 Unit Testing 下的子话题。它给的能力边界也很具体：`@nuxt/test-utils` 提供**可选的对等依赖**，运行时环境在 `happy-dom` 与 `jsdom` 之间选、端到端 runner 在 `vitest`/`cucumber`/`jest`/`playwright` 之间选、`playwright-core` 只在你想用内置浏览器测试工具且不拿 `@playwright/test` 当 runner 时才需要；而**单元测试环境目前只支持 `vitest`**（官方括注"欢迎贡献其他 runtime"）。这一篇先把分层与选择讲清，配置细节留给 491。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - test-utils
  - vitest
  - playwright
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

你按"单元、组件、集成、端到端"四层规划测试目录，翻到官方文档发现对不上——那一节只有两大类，而且**其中一类的 runner 只支持一个**。

这一篇先把官方给的分层与可选依赖列清楚，避免目录结构定完才发现落不了地。

## 定位：一套 test-utils，两类测试

官方第一段的用词值得照抄：

> Nuxt offers first-class support for end-to-end and unit testing of your Nuxt application via `@nuxt/test-utils`, a library of test utilities and configuration that currently powers the tests we use on Nuxt itself and tests throughout the module ecosystem.

三件事：**支持的类型是"端到端"与"单元"两种**、载体是 `@nuxt/test-utils` 这个库（工具加配置）、可信度线索是"Nuxt 自己的测试与模块生态的测试都在用它"。

整页的高层结构就是三节：**Installation、Unit Testing、End-To-End Testing**。Unit Testing 之下再分子话题（如何组织、内置 mock、helpers、`@vue/test-utils`、Vitest 浏览器模式）。所以：

| 常被说的分层 | 在 4.x 文档里的位置 |
| --- | --- |
| 单元测试 | Unit Testing 一节 |
| 组件测试 | **Unit Testing 的子话题**（`@vue/test-utils` 与 Nuxt 环境那一套），不是独立一层 |
| 集成测试 | **文档里没有这一层**（未记载这个词） |
| 端到端测试 | End-To-End Testing 一节 |

## 依赖怎么选

安装段的理由是这一句：

> In order to allow you to manage your other testing dependencies, `@nuxt/test-utils` ships with various optional peer dependencies.

三条可选面官方列出来了：

- 运行时环境：`happy-dom` 与 `jsdom` 二选一
- 端到端 runner：`vitest`、`cucumber`、`jest`、`playwright` 四选一
- `playwright-core`：**只在**你想用内置浏览器测试工具、且不把 `@playwright/test` 当 runner 时才需要

官方给的这一行安装命令把三样一起装了：

```bash [npm]
npm i --save-dev @nuxt/test-utils vitest @vue/test-utils happy-dom playwright-core
```

（同一块还给了 yarn / pnpm / bun 三种写法，内容一致。）注意这里出现了 `@vue/test-utils`——它不在上面那三条列举里，却出现在官方安装命令中，因为组件测试那一节要用它。

## 两个硬限定

**第一条：单元环境只支持 vitest。** 原文：

> We currently ship an environment for unit testing code that needs a Nuxt runtime environment. It currently *only has support for `vitest`* (although contribution to add other runtimes would be welcome).

一句里两个信号：`only has support for vitest` 加"欢迎贡献"。**想在 jest 下跑需要 Nuxt 运行时的单元测试，官方没有提供这条路**。

**第二条：端到端支持四个 runner。** 那句话在 E2E 一节开头："For end-to-end testing, we support Vitest, Jest, Cucumber and Playwright as test runners."，与前面的四选一对得上。**注意方向**：e2e 四个都行，Nuxt 运行时的单元只有一个。

## 两种配法，官方推荐哪一种

Unit Testing 的 Setup 是三步：

1. 把 `@nuxt/test-utils/module` 加进 `nuxt.config`（**可选**，它加的是 Nuxt DevTools 里的 Vitest 集成，用来在开发时跑单元测试）
2. 建 `vitest.config.ts`，用 projects 把 `unit` / `e2e` / `nuxt` 三个环境分开
3. Nuxt 环境的测试文件若不在 `test/nuxt/` 下，去看 TypeScript Support in Tests 那节把它们纳入类型上下文

第三步之前有两条 `::tip`，都属于"配不上就先查这里"：

> When importing `@nuxt/test-utils` in your vitest config, it is necessary to have `"type": "module"` specified in your `package.json` or rename your vitest config file appropriately. i.e., `vitest.config.m{ts,js}`.

> It is possible to set environment variables for testing by using the `.env.test` file.

第一条能解释一个常见报错：**在 vitest 配置里 import 这个库，`package.json` 就得有 `"type": "module"`**，或者把配置文件改名成 `vitest.config.mts`/`.mjs`。

配置的具体形状、每个 environment 的含义留给 491 那篇。

## 一句组织原则

官方在"如何组织测试"那节给的目录是 `test/e2e/`、`test/nuxt/`、`test/unit/` 三个，收尾那句是这一篇最该记住的约束：

> You can of course opt for any test structure, but keeping the Nuxt runtime environment separated from Nuxt end-to-end tests is important for test stability.

**目录形状随你定，但"需要 Nuxt 运行时的那批"与"端到端那批"要分开**——官方给的理由是测试稳定性。

## 自查

- 4.x 只有两类：单元与端到端；"集成测试"这个词文档没用，组件测试是单元那节下的子话题。
- 可选对等依赖三条：dom 环境（happy-dom/jsdom）、e2e runner（vitest/cucumber/jest/playwright）、`playwright-core`（内置浏览器工具且不换 `@playwright/test` 时才装）。
- 需要 Nuxt 运行时的单元测试**只支持 vitest**；e2e 四个 runner 都行。
- 在 vitest 配置里引这个库，要么 `package.json` 有 `"type": "module"`，要么文件名换成 `.mts`/`.mjs`。
- 测试用环境变量放 `.env.test`；目录形状自由，但 Nuxt 运行时与 e2e 必须分开。

## 下一篇

下一篇 491 把 `vitest.config.ts` 那份原文配置逐行拆开：三个 project 各自的 `environment`（`unit` 与 `e2e` 都是 `node`，只有 `nuxt` 是 `environment: 'nuxt'` 且必须用 `defineVitestProject`），以及官方那条 note——`defineVitestProject` **只给 Nuxt 环境的测试用**，端到端要按普通 `test.environment: 'node'` 的 project 配。
