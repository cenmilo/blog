---
url: /posts/1d4619e80160d3a57b962699361491a1/
title: 508 - Vitest Browser Mode：配置四行与 render 的八个返回项
date: 2027-03-30T10:00:00+08:00
updated: 2027-03-30T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：第三种执行环境：Vitest 自己的 Browser Mode，`@nuxt/test-utils` 为它提供了 `@nuxt/test-utils/browser`。配置四行——`name: 'browser'`、`include: ['test/browser/**/*.{test,spec}.ts']`、`browser: { enabled: true, provider: playwright(), instances: [{ browser: 'chromium' }] }`、以及可选的 `setupFiles: ['@nuxt/test-utils/browser']`（加了它才有 `page.render` 与自动清理）。用法有两条入口：从 `vitest/browser` 拿 `page.render`，或直接从 `@nuxt/test-utils/browser` 拿 `render`。选项里有一条明确的替换要求：**用 `container` 而不是 Vue Test Utils 的 `attachTo`**。返回值有八项，含 `locator`、`debug()`、`emitted()`、`rerender(props)`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - vitest
  - 浏览器
  - browser-mode
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

到这里你已经见过两种执行环境：jsdom/happy-dom 里挂组件，以及 e2e 那侧起一个真浏览器。测试页里还有第三种——**浏览器由 Vitest 自己开**，而不是由 Nuxt 的测试工具开。

官方一句话定性：**"@nuxt/test-utils provides helpers for testing in Vitest Browser Mode via `@nuxt/test-utils/browser`."**

## 配置：一个 project，四行关键内容

第一步装包，`@vitest/browser-playwright` 一个（npm/yarn/pnpm/bun 四条命令，这一节同样没有 deno）：

```bash [npm]
npm i --save-dev @vitest/browser-playwright
```

第二步把 browser project 加进 `vitest.config.ts` 的 `projects` 里，官方整份示例：

```ts
import { defineConfig } from 'vitest/config'
import { defineVitestProject } from '@nuxt/test-utils/config'
import { playwright } from '@vitest/browser-playwright'

export default defineConfig({
  test: {
    projects: [
      await defineVitestProject({
        test: {
          name: 'browser',
          include: ['test/browser/**/*.{test,spec}.ts'],
          browser: {
            enabled: true,
            provider: playwright(),
            instances: [{ browser: 'chromium' }],
          },
          // If you want to enable `page.render` and automatic cleanup, add this setup file
          setupFiles: ['@nuxt/test-utils/browser'],
        },
      }),
    ],
  },
})
```

四行各自的含义：`name: 'browser'` 是这个 project 的名字；`include` 把它绑到 `test/browser/` 目录、匹配 `.test.ts` 与 `.spec.ts` 两种后缀；`browser` 那三行是 Vitest Browser Mode 本身（开、用 Playwright 作 provider、跑一个 chromium 实例）；`setupFiles` 是可选的，注释写得很直白——**加了这个文件才有 `page.render` 和自动清理**。

注意它仍然包在 `defineVitestProject` 里，也就是说这是**走 Nuxt 环境的那个包装函数**，不是普通 Vitest project。

两条 `::note` 都是关于类型认不认得出来：

- 把 `@nuxt/test-utils/browser` 放进 `setupFiles` 后，如果 TypeScript 还是不认 `page.render` 的类型，可以在 `nuxt.config.ts` 里通过 `typescript.tsConfig.compilerOptions.types` 加上它。
- 浏览器模式的测试如果不在 `test/nuxt/` 而在 `test/browser/`（示例正是这样），要按 "Adding other test directories" 那节把它们加进 TypeScript 上下文。

第二条很容易被忽略：`include` 决定了**跑不跑**，TypeScript 上下文决定**编辑器认不认**。两处都配上，才既绿又不飘红。

## 两种入口，同一套断言

第一条路用 `page.render`：

```ts [test/browser/components/MyCounter.nuxt.spec.ts]
import { expect, it } from 'vitest'
import { page } from 'vitest/browser'
// If you added the setup file, the following import is not needed
import '@nuxt/test-utils/browser'

import { MyCounter } from '#components'

it('counter button increments the count', async () => {
  const screen = await page.render(MyCounter)
  await screen.getByRole('button', { name: 'Increment' }).click()
  await expect.element(screen.getByText('Count: 1')).toBeVisible()
})
```

第二条路直接引 `render`：

```ts
import { expect, it } from 'vitest'
import { render } from '@nuxt/test-utils/browser'
import { MyCounter } from '#components'

it('can render using the render helper', async () => {
  const screen = await render(MyCounter)
  await screen.getByRole('button', { name: 'Increment' }).click()
  await expect.element(screen.getByText('Count: 1')).toBeVisible()
})
```

两个细节值得单独说。**组件仍然从 `#components` 具名导入**——这条路还是 Nuxt 环境。**断言是 `await expect.element(...).toBeVisible()`**，这是 Vitest 浏览器模式的断言形式，与前面见过的 `expect(wrapper.text())`、`screen.getByText(...)` 都不是一个 API。

## 选项四项、返回八项

选项那段的说明有一句是替换要求：**"The options object accepts `@vue/test-utils` mount options (use `container` instead of `attachTo`)"**。

| 选项 | 官方说明 | 默认 |
| --- | --- | --- |
| `route` | the initial route, or `false` to skip the initial route change | `/` |
| `spy` | enable spy component setup state（示例指回 `mountSuspended`） | `false` |
| `container` | 自定义渲染进去的 `HTMLElement` 容器，**用它替代 Vue Test Utils 的 `attachTo`** | 文档未给 |
| `baseElement` | 自定义基准 `HTMLElement` | `document.body` |

`route` 与 `spy` 的措辞与 `mountSuspended`／`renderSuspended` 逐字相同——三处共用一套语义。

返回那段只有一句前提（"The return object includes the following properties"），八项：

| 返回项 | 官方说明 |
| --- | --- |
| `container` | 组件渲染进去的那个容器 `HTMLElement` |
| `baseElement` | 基准 `HTMLElement`（默认 `document.body`） |
| `locator` | 根元素的 `Locator` |
| `setupState` | 组件 setup 的返回值；**开了 `spy` 时它是被 mock 过的** |
| `debug()` | 把格式化后的 DOM 打到控制台 |
| `unmount()` | 卸载组件，同时记录一个 `nuxt.unmount` trace mark |
| `emitted()` | 取已发出的事件 |
| `rerender(props)` | 用新 props 重渲染，同时记录一个 `nuxt.rerender` trace mark |

四项是值、四项是方法。`setupState` 那行的措辞与其他两处不同：挂载类 helper 说它是 setup 的返回，这里补了"开 `spy` 时被 mock"——也就是 `spy` 打开后它变成可替换实现的形态。`debug()` 与 `emitted()` 是排错时最短的两条路：一个看真实 DOM，一个看事件到底发没发。**文档没写这两个方法的参数与输出格式。**

## 自查

- 这是第三种执行环境：浏览器由 Vitest Browser Mode 开，provider 用 `@vitest/browser-playwright`，但仍包在 `defineVitestProject` 里，组件照样从 `#components` 导入。
- `setupFiles: ['@nuxt/test-utils/browser']` 是可选的，加它才有 `page.render` 与自动清理；类型不认时再用 `typescript.tsConfig.compilerOptions.types`。
- 目录在 `test/browser/` 时，`include` 与 TypeScript 上下文两处都要配。
- 选项四个：`route`／`spy`／`container`／`baseElement`，其中 `container` **替代** `attachTo`。
- 返回八项，`setupState` 在 `spy` 开启时是被 mock 的；`unmount()` 与 `rerender()` 会各自记录一个 trace mark。

## 下一篇

下一篇 509 把端到端这一侧的方法面收齐：三个取页面的方法各自返回什么、`createPage` 与 Playwright 运行器与 Browser Mode 三种浏览器入口怎么选。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/testing（Vitest Browser Mode 整节：定性句、Setup 两步（`@vitest/browser-playwright` 四包管理器安装、`projects` 内的 `defineVitestProject` 配置全文含 `name`/`include`/`browser.enabled`/`provider`/`instances`/`setupFiles` 与那条注释）、两条 `::note`（`compilerOptions.types` 与 test/browser/ 目录要加进 TS 上下文）、Usage 的两种入口示例与 `expect.element(...).toBeVisible()`、选项四项与返回八项的逐条说明含 trace mark）
- https://nuxt.com/docs/4.x/getting-started/testing（对照：`mountSuspended`/`renderSuspended` 的 `route`/`spy`/`setupState` 措辞、Adding other test directories 小节）
