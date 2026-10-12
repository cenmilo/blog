---
url: /posts/01cdc35308909d21f16653c9f508ae19/
title: test-utils 的能力边界（照文档记载）
date: 2027-03-25T09:00:00+08:00
updated: 2027-03-25T09:00:00+08:00
author: cmdragon

summary:
  摘要：这套工具由五个子路径组成，官方文档里出现的入口是 `@nuxt/test-utils/config`（`defineVitestProject`、`defineVitestConfig`）、`/runtime`（`mountSuspended`、`renderSuspended`、`mockNuxtImport`、`mockComponent`、`registerEndpoint`）、`/e2e`（`setup`、`$fetch`、`fetch`、`url`、`getServerLogs`、`clearServerLogs`、`createPage`）、`/browser`（`render`），以及可选的 `/module`（DevTools 里的 Vitest 集成）。**一条硬边界写在 Helpers 末尾：`/runtime` 与 `/e2e` 需要跑在不同测试环境里，因此不能出现在同一个文件中**。内置 mock 只有两个（`intersectionObserver` 默认 `true`、给的是一个无任何功能的假类；`indexedDB` 默认 `false`、开了才用 fake-indexeddb 给可用的模拟，而配置键写作 `indexedDb`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - test-utils
  - mock
  - 端到端
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一个测试文件里，你从 `@nuxt/test-utils/runtime` 引了 `mountSuspended`，又从 `@nuxt/test-utils/e2e` 引了 `$fetch`——报错信息还看不太懂。

官方在文档里明确写了这一条不允许。这一篇把这套工具有哪些入口、各自能干什么、边界划在哪，按记载列一遍。

## 五个子路径

官方文档里实际出现过的导入路径有五个：

| 子路径 | 文档里出现过的导出 | 用在哪 |
| --- | --- | --- |
| `/config` | `defineVitestProject`、`defineVitestConfig` | `vitest.config.ts`（491 那份三 project 配置） |
| `/runtime` | `mountSuspended`、`renderSuspended`、`mockNuxtImport`、`mockComponent`、`registerEndpoint` | Nuxt 环境的单元测试（`test/nuxt/`） |
| `/e2e` | `setup`、`$fetch`、`fetch`、`url`、`getServerLogs`、`clearServerLogs`、`createPage` | 端到端测试（`test/e2e/`） |
| `/browser` | `render` | Vitest 浏览器模式那一节 |
| `/module` | —— | 可选加进 `nuxt.config`，给 DevTools 加 Vitest 集成 |

## 最硬的一条边界

Helpers 一节末尾有个小标题就叫 "Conflict with End-To-End Testing"，原文一句：

> `@nuxt/test-utils/runtime` and `@nuxt/test-utils/e2e` need to run in different testing environments and so can't be used in the same file.

**`/runtime` 与 `/e2e` 不能出现在同一个文件里**，理由是两个要跑在不同的测试环境下。开头那个现象就是这条——规则本身就一句，报错看不懂不要紧。

这也回指 491 那份配置为什么要按 project 分环境：**文件属于哪个子路径，就得落进对应的 environment**。

## 内置 mock 只有两个

Built-In Mocks 一节的定性句是"`@nuxt/test-utils` provides some built-in mocks for the DOM environment"，列出来的只有两项：

| mock | 默认 | 官方说明 |
| --- | --- | --- |
| `intersectionObserver` | **`true`** | creates a dummy class **without any functionality** for the IntersectionObserver API |
| `indexedDB` | **`false`** | uses `fake-indexeddb` to create a **functional mock** of the IndexedDB API |

两个默认值的含义要读准：**IntersectionObserver 默认就有，但它是个没有任何功能的空壳**（只是不报"未定义"，不会真的触发回调）；**IndexedDB 默认没有**，要能用的模拟得自己打开。配置位置在 `vitest.config.ts` 的 `environmentOptions` 里：

```ts twoslash
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    environmentOptions: {
      nuxt: {
        mock: {
          intersectionObserver: true,
          indexedDb: true,
        },
      },
    },
  },
})
```

**键名不对称要留意**：小标题写的是 `indexedDB`（大写 DB），配置对象里写的是 `indexedDb`。照标题抄进配置会静默不生效——官方没解释这处不一致（文档未记载），**以示例里的 `indexedDb` 为准**。

## 五个 helper 各自的范围

**`mountSuspended`** —— 官方定义：

> `mountSuspended` allows you to mount any Vue component within the Nuxt environment, allowing async setup and access to injections from your Nuxt plugins.

两个能力点：**支持 async setup**、**能拿到你在 Nuxt 插件里 provide 的注入**。实现方式那条 `::note` 也写明：它内部包的**就是 `@vue/test-utils` 的 `mount`**，所以能传哪些选项、怎么用，去看 Vue Test Utils 文档。

**`renderSuspended`** —— 换的是库：

> `renderSuspended` allows you to render any Vue component within the Nuxt environment using `@testing-library/vue`, allowing async setup and access to injections from your Nuxt plugins.

配套三条要求不做就用不起来：要**自己装 `@testing-library/vue`** 才配得上 `screen`、`fireEvent` 这些工具；Testing Library 依赖 testing globals 做清理，**要在 Vitest 配置里把 globals 打开**；渲染位置固定——**组件被放进 `<div id="test-wrapper"></div>`**（这条决定你查 DOM 时的根节点）。

**`mockNuxtImport`** 与 **`mockComponent`** 各占一节，用来替换自动导入的函数与组件（具体用法在 493 那篇展开）。

**`registerEndpoint`** 是造 Nitro 假接口的，参数与默认值写得清楚：

> The first argument is the endpoint name (e.g. `/test/`). The second argument is a factory function that returns the mocked data.

```ts twoslash
import { registerEndpoint } from '@nuxt/test-utils/runtime'

registerEndpoint('/test/', () => ({
  test: 'test-field',
}))
```

重要默认值一句：**"By default, your request will be made using the `GET` method."** 要换方法就得把第二个参数改成对象：

```ts twoslash
import { registerEndpoint } from '@nuxt/test-utils/runtime'

registerEndpoint('/test/', {
  method: 'POST',
  handler: () => ({ test: 'test-field' }),
})
```

对象接受三个属性：`handler`（事件处理函数）、`method`（可选，要匹配的 HTTP 方法）、`once`（可选，为 true 时**该处理器只服务第一个匹配请求，之后自动移除**）。

还有一条 Note 处理"组件请求的是外部 API"：用 `baseURL`，再通过**环境覆盖配置里的 `$test`** 把它设成空，于是所有请求都走 Nitro 服务端——也就是上面这个假接口。

## e2e 那侧的 API 形状

| API | 官方定义 |
| --- | --- |
| `$fetch(url)` | Get the HTML of a server-rendered page. |
| `fetch(url)` | Get the response of a server-rendered page.（示例里取的是 `res.body` 与 `res.headers`） |
| `url(path)` | Get the full URL for a given page，**含测试服务端跑着的端口**（示例结果形如 `http://localhost:6840/page`） |
| `getServerLogs()` | 返回自上次 `startServer()`（或 `clearServerLogs()`）以来从服务端子进程 stdout/stderr 捕获的行；**只有 `captureServerLogs` 为 `true`（默认）时才有内容** |
| `clearServerLogs()` | 清掉已捕获的服务端日志行；官方说这在"只想断言某一次操作产生的日志"时有用 |

`getServerLogs()` 那条官方示例的组合是：`clearServerLogs()` → `$fetch('/api/log-test')` → 用 `vi.waitFor` 轮询直到日志里出现标记串。**这是文档里唯一一处"从测试里断言服务端行为"的写法**——458 讲性能回归时说过，这条通道只到日志层，没有指标。

浏览器那侧官方一句话定性："We provide built-in support using Playwright within `@nuxt/test-utils`, either programmatically or via the Playwright test runner."，用法示例（`createPage('/login')` 配 `page.getByTestId(...)`）在 494 那篇展开。

## 自查

- 五个子路径各管一段；**`/runtime` 与 `/e2e` 不能同文件**，这条写在 Helpers 末尾的小节里。
- 内置 mock 只有 `intersectionObserver`（默认开、但是**无功能的空壳**）与 `indexedDB`（默认关、开了才是可用模拟），配置键写作 `indexedDb`。
- `mountSuspended` 包的是 `@vue/test-utils` 的 `mount`；`renderSuspended` 要自装 `@testing-library/vue`、开 Vitest globals、渲染进 `<div id="test-wrapper">`。
- `registerEndpoint` 默认 GET，改方法要传对象（`handler`/`method`/`once`）；外部 API 用 `$test` 覆盖 `baseURL` 引到 Nitro。
- e2e 五个 API 里 `getServerLogs` 依赖 `captureServerLogs` 为真，断言服务端行为要靠 `vi.waitFor` 轮询。

## 下一篇

下一篇 493 讲组件测试的官方用法：`mountSuspended` 与 `renderSuspended` 各自适合什么写法、示例里那个从 `#components` 拿具名导出的做法、以及三份官方示例分别断言了什么。

## 本文依据

- 测试页 Installation 三条可选对等依赖与 `/module`（DevTools 里的 Vitest 集成）、`/config` 两个函数（`defineVitestProject`、`defineVitestConfig`）与 `environmentOptions.nuxt` 的 `rootDir`/`domEnvironment`/`overrides` 注释（`getting-started/testing`）
- Built-In Mocks 一节：`intersectionObserver`（默认 `true`、dummy class without any functionality）、`indexedDB`（默认 `false`、fake-indexeddb 的 functional mock）、`environmentOptions.nuxt.mock` 配置示例（键名 `indexedDb`）（`getting-started/testing`）
- Helpers 一节：`mountSuspended` 定义句与"wraps `mount` from `@vue/test-utils`"的 `::note`、`renderSuspended` 定义句（用 `@testing-library/vue`、需自装、需开 Vitest globals、渲染进 `<div id="test-wrapper"></div>`）、`mockNuxtImport` 与 `mockComponent` 两节、`registerEndpoint` 的两个参数说明／默认 GET／对象形态与 `handler`/`method`/`once` 三属性／指向环境覆盖 `$test` 的 Note、"Conflict with End-To-End Testing"整句（`getting-started/testing`）
- E2E APIs 一节：`$fetch`/`fetch`/`url`/`getServerLogs`/`clearServerLogs` 五段定义与示例（含 `http://localhost:6840/page`、`captureServerLogs` 前提、`vi.waitFor` 用法）、Testing in a Browser 的 Playwright 定性句与 `createPage` 示例；Vitest 浏览器模式的 `render` from `@nuxt/test-utils/browser`（`getting-started/testing`）
