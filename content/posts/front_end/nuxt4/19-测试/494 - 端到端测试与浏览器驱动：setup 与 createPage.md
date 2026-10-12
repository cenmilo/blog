---
url: /posts/0cc42c43707f658cedd1323e07720b54/
title: 端到端测试与浏览器驱动：setup 与 createPage
date: 2027-03-25T11:00:00+08:00
updated: 2027-03-25T11:00:00+08:00
author: cmdragon

summary:
  摘要：组件测试跑在 happy-dom 或 jsdom 里，那不是浏览器——水合时序、真实事件循环、CSS 相关的行为都测不到。端到端这一侧官方支持 Vitest、Jest、Cucumber、Playwright 四种运行器，入口是 `@nuxt/test-utils/e2e` 的 `setup()`：它把「Nuxt 配置、时间、功能」三组共十条选项在 `beforeAll`/`beforeEach`/`afterEach`/`afterAll` 里替你做事，其中 `build` 与 `server` 的默认值会随 `browser`、`host` 变化。浏览器有两种驱动方式：在测试里 `createPage('/page')` 直接拿 Playwright 的 `page`，或者用 Playwright 自己的测试运行器，从 `@nuxt/test-utils/playwright` 引 `expect` 与 `test`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 端到端
  - playwright
  - test-utils
  - 测试
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

组件测试一片绿，页面在真浏览器里还是坏的。这不是你的测试写错了——是环境不同。文档在 Running Tests 那条 `::warning` 里写得很清楚：在 Nuxt 环境里跑的测试，环境是 **happy-dom 或 jsdom**，测试运行前会初始化一个全局 Nuxt 应用。jsdom 不是浏览器，浏览器接手 HTML 并挂上事件的那一段（也就是水合）在你的测试里根本没有发生。

要覆盖这一段，就得换到端到端这一侧。

## 四种运行器与一句话前提

> For end-to-end testing, we support Vitest, Jest, Cucumber and Playwright as test runners.

`setup` 相关的说明里还有一句：「Behind the scenes, `setup` performs a number of tasks in `beforeAll`, `beforeEach`, `afterEach` and `afterAll` to set up the Nuxt test environment correctly.」——它不是单纯启动一下，而是把这几钩子都占了。所以每个用到 `@nuxt/test-utils/e2e` 里那些方法的 `describe` 块，都要在块内先 `await setup({...})`：

```ts [test/my-test.spec.ts]
import { describe, test } from 'vitest'
import { $fetch, setup } from '@nuxt/test-utils/e2e'

describe('My test', async () => {
  await setup({
    // test context options
  })

  test('my test', () => {
    // ...
  })
})
```

## setup 的十条选项，逐条默认值

文档按三组列，标题分别是 Nuxt Config、Timings、Features。

| 组 | 选项 | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- | --- |
| Nuxt Config | `rootDir` | 被测试的 Nuxt 应用所在目录 | `string` | `'.'` |
| Nuxt Config | `configFile` | 配置文件名 | `string` | `'nuxt.config'` |
| Timings | `setupTimeout` | 允许 `setupTest` 完成工作的毫秒数（按传进来的选项，这可能包含构建或生成文件） | `number` | `120000`，Windows 上 `240000` |
| Timings | `teardownTimeout` | 允许拆掉测试环境的毫秒数，比如关浏览器 | `number` | `30000` |
| Features | `build` | 是否跑单独一步构建 | `boolean` | `true`（`browser` 或 `server` 关掉时、或提供了 `host` 时为 `false`） |
| Features | `server` | 是否在测试套件里启动一个响应请求的服务器 | `boolean` | `true`（提供了 `host` 时为 `false`） |
| Features | `port` | 提供了就把启动的测试服务端口设成该值 | `number \| undefined` | `undefined` |
| Features | `host` | 提供了就用这个 URL 作为测试目标，而不是构建并新起服务器 | `string` | `undefined` |
| Features | `browser` | 置为真则启动一个浏览器并在后续测试里控制它 | `boolean` | `false` |
| Features | `runner` | 指定测试套件用的运行器 | `'vitest' \| 'jest' \| 'cucumber'` | `'vitest'`（文档注明"Currently, Vitest is recommended"） |
| Features | `logLevel` | 覆盖服务端子进程的 consola 日志级别 | `number` | `1`（可用 `NUXT_TEST_LOG_LEVEL` 环境变量覆盖） |
| Features | `captureServerLogs` | 是否捕获服务端进程输出而非继承 stdio | `boolean` | `true` |

`browserOptions` 是对象，文档列出两个属性：`type`（`chromium`、`firefox`、`webkit` 三选一）、`launch`（启动浏览器时传给 Playwright 的选项）。

三件事值得停下来看：

**`host` 会连带改掉两个默认值。** 一旦给了 `host`，`build` 与 `server` 都变 `false`——官方对它的定位是"跑真正端到端、对着已部署版本或已在本地跑着的服务器"，并明说这能显著缩短测试时间。文档给的例子是登录页：

```ts
import { createPage, setup } from '@nuxt/test-utils/e2e'
import { describe, expect, it } from 'vitest'

describe('login page', async () => {
  await setup({
    host: 'http://localhost:8787',
  })

  it('displays the email and password fields', async () => {
    const page = await createPage('/login')
    expect(await page.getByTestId('email').isVisible()).toBe(true)
    expect(await page.getByTestId('password').isVisible()).toBe(true)
  })
})
```

**`captureServerLogs` 关掉是调试用的。** 原文：置 `false` 恢复旧的继承 stdio 行为，"useful when debugging a test locally"——因为默认 `true` 时服务端的输出被从控制台压掉，只能靠 `getServerLogs()` 读。

**`config` 这项现在用不了。** 文档里 "Object with configuration overrides" 那一条整段被注释掉了，没有作为可传选项列出。要覆盖配置，走 `rootDir` 指向的那个应用自己的 `nuxt.config`。

## 拿 HTML 与拿响应

`$fetch(url)` 与 `fetch(url)` 是这一侧最常用的两个：前者「Get the HTML of a server-rendered page」，后者拿响应对象，示例里取的是 `res.body` 与 `res.headers`。加一个 `url(path)`，返回含测试服务端口的完整地址（示例结果形如 `http://localhost:6840/page`）。

断言服务端行为只有一条通道：`getServerLogs()`。文档唯一的例子标题就是 `captures console.log output from a server route`，组合是 `clearServerLogs()` → `$fetch('/api/log-test')` → 用 `vi.waitFor` 轮询到日志里出现标记串。

## 浏览器：编程式与 Playwright 运行器

> We provide built-in support using Playwright within `@nuxt/test-utils`, either programmatically or via the Playwright test runner.

编程式只有一个入口：

```ts
import { createPage } from '@nuxt/test-utils/e2e'

const page = await createPage('/page')
// you can access all the Playwright APIs from the `page` variable
```

官方说明是：在 `vitest`、`jest` 或 `cucumber` 里用它创建一个配置好的 Playwright 浏览器实例，可以（可选地）指向运行中服务器的某个路径；`page` 上有哪些方法去看 Playwright 文档。

另一条路是直接用 Playwright 的测试运行器。装两个包：

```bash [npm]
npm i --save-dev @playwright/test @nuxt/test-utils
```

yarn／pnpm／bun／deno 各有一条对应命令（`yarn add --dev`、`pnpm add -D`、`bun add --dev`、`deno add --dev npm:@playwright/test npm:@nuxt/test-utils`）。

全局配置写在 `playwright.config.ts`：

```ts [playwright.config.ts]
import { fileURLToPath } from 'node:url'
import { defineConfig, devices } from '@playwright/test'
import type { ConfigOptions } from '@nuxt/test-utils/playwright'

export default defineConfig<ConfigOptions>({
  use: {
    nuxt: {
      rootDir: fileURLToPath(new URL('.', import.meta.url)),
    },
  },
  // ...
})
```

关键在那句「with the same configuration details as the `setup()` function mentioned earlier」：`ConfigOptions` 里能写的就是上面 `setup` 那十条，只是位置从测试文件挪到了配置的 `use.nuxt`。

测试文件的写法有个反直觉的地方——`expect` 和 `test` 不从 `@playwright/test` 引：

```ts [tests/example.test.ts]
import { expect, test } from '@nuxt/test-utils/playwright'

test('test', async ({ page, goto }) => {
  await goto('/', { waitUntil: 'hydration' })
  await expect(page.getByRole('heading')).toHaveText('Welcome to Playwright!')
})
```

`goto` 是这套扩展给的，不是 Playwright 自带。`waitUntil: 'hydration'` 这个取值在文档里只出现在示例中，没有选项表说明它等待的具体信号——想确认行为只能自己跑一次对比。

不想动配置文件也可以，直接在测试文件里声明 Nuxt 服务器：

```ts [tests/example.test.ts]
import { expect, test } from '@nuxt/test-utils/playwright'

test.use({
  nuxt: {
    rootDir: fileURLToPath(new URL('..', import.meta.url)),
  },
})

test('test', async ({ page, goto }) => {
  await goto('/', { waitUntil: 'hydration' })
  await expect(page.getByRole('heading')).toHaveText('Welcome to Playwright!')
})
```

两种写法的断言部分逐字相同，差别只有 `rootDir` 的相对层级。

## 自查

- 组件测试的环境是 happy-dom 或 jsdom，水合不发生；要测真实浏览器就得走 e2e。
- `setup` 占了四个钩子，所以必须在每个用到 e2e 方法的 `describe` 里先 `await`；每个测试文件只能引 `/runtime` 或 `/e2e` 其中一个。
- 十条选项里三条会互相牵连：给 `host` → `build`、`server` 变 `false`；关 `browser` 或 `server` → `build` 也变 `false`；`setupTimeout` 在 Windows 上是 `240000`。
- `config` 覆盖项在文档里是注释掉的，别当成可用参数。
- Playwright 运行器模式下 `expect`/`test` 从 `@nuxt/test-utils/playwright` 引，配置通过 `ConfigOptions` 或 `test.use({ nuxt })` 传，等待水合用扩展的 `goto`。

## 下一篇

下一篇 495 讲 `nuxt test` 这条命令：它带的两个参数、三个选项，以及它对环境变量做的那件事。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/testing（End-To-End Testing：四种运行器句、`setup` 的四个钩子说明与最小示例、Nuxt Config/Timings/Features 三组选项全部条目与默认值、`browserOptions` 的 `type`/`launch`、被注释掉的 `config`、Target `host` 小节与登录页示例）
- https://nuxt.com/docs/4.x/getting-started/testing（APIs 一节 `$fetch`/`fetch`/`url`/`getServerLogs`/`clearServerLogs`；Testing in a Browser 的 `createPage` 定义与 Playwright Test Runner 的安装、`playwright.config.ts`、`@nuxt/test-utils/playwright` 的 `expect`/`test`、`test.use` 两种配置写法）
- https://nuxt.com/docs/4.x/getting-started/testing（Running Tests 的 `::warning`：happy-dom 或 jsdom、测试前初始化全局 Nuxt 应用）
