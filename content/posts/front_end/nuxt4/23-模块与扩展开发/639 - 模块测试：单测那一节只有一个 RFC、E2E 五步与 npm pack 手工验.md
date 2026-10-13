---
url: /posts/65ace43aac9dc381548f8835343701d9/
title: 639 - 模块测试：单测那一节只有一个 RFC、E2E 五步与 npm pack 手工验
date: 2027-05-13T09:00:00+08:00
updated: 2027-05-13T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：模块测试那一节有三个小节，**第一个小节是空的**——"写单元测试"整节只有一条 `::tip`，内容是官方仍在讨论怎么让模块的单元与集成测试更省事，并给了一个 RFC 讨论链接（nuxt/nuxt 讨论 18399）。真正有做法的是另外两节：**E2E 用 Nuxt Test Utils，五步流程是"在 `test/fixtures/*` 里建一个当 fixture 的 Nuxt 应用 → 在测试文件里 `setup` 它 → 用 `@nuxt/test-utils` 的工具与之交互 → 对这个 fixture 做断言 → 重复"**；fixture 的配置里直接 `modules: [MyModule]` 引源码，所以改 `src` 立刻被测到。**手工验证走 `npm pack` 出 tarball、再用 `file:` 装进另一个项目**，这是唯一一条能验"发出去的形态"的路。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块测试
  - e2e
  - test-utils
  - npm-pack
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块的测试和应用的测试不是一回事：你要验的是"别人的项目装上我的模块之后会怎样"。这一节给了三条路，**其中一条现在还没有内容**。

## 三条路的分工

| 小节 | 这一节给了什么 | 能不能照着做 |
| --- | --- | --- |
| Write Unit Tests | 只有一条 tip ＋ 一个 RFC 链接 | 不能，官方写明还在讨论 |
| Write E2E Tests | 五步流程 ＋ fixture 与测试两份完整代码 | 能 |
| Test Manually | `npm pack` 出一个 tarball、用 `file:` 装进另一个项目 | 能 |

## 单测那一节只有一段讨论邀请

整节的原文是一条 `::tip`：

> We're still discussing and exploring how to ease unit and integration testing on Nuxt modules.

后面接一句 "[Check out this RFC to join the conversation](https://github.com/nuxt/nuxt/discussions/18399)"。

这条信息本身有用：**别在文档里找"模块单测的标准写法"**，官方把它标为未定。也就是说模块内部纯逻辑的部分要测，得自己按 Vitest 的常规方式安排；而**"需要 Nuxt 运行时才能验的行为"这一节给的答复是走 E2E**。

## E2E 的五步

定性一句：**Nuxt Test Utils 是你端到端地测试模块要用的库**。流程是五步：

1. 在 `test/fixtures/*` 里创建一个 Nuxt 应用作为 "fixture"
2. 在你的测试文件里用这个 fixture 来 setup Nuxt
3. 用 `@nuxt/test-utils` 提供的工具与 fixture 交互（例如取一个页面）
4. 针对这个 fixture 做检查（例如"HTML 里包含 ……"）
5. 重复

fixture 侧的代码：

```ts [test/fixtures/ssr/nuxt.config.ts]
// 1. Create a Nuxt application to be used as a "fixture"
import MyModule from '../../../src/module'

export default defineNuxtConfig({
  ssr: true,
  modules: [
    MyModule,
  ],
})
```

**注意第 3 行：`modules` 里放的不是包名，而是从 `../../../src/module` 直接 import 进来的模块对象**。这正是 `modules` 数组四种形状里的"内联模块定义"，效果是**测试跑的永远是源码当前状态**，不需要先构建、也不需要先打包。fixture 本身就是一个普通 Nuxt 配置，这里显式开了 `ssr: true`。

测试侧：

```ts [test/rendering.ts]
import { describe, expect, it } from 'vitest'
import { fileURLToPath } from 'node:url'
import { $fetch, setup } from '@nuxt/test-utils/e2e'

describe('ssr', async () => {
  // 2. Setup Nuxt with this fixture inside your test file
  await setup({
    rootDir: fileURLToPath(new URL('./fixtures/ssr', import.meta.url)),
  })

  it('renders the index page', async () => {
    // 3. Interact with the fixture using utilities from `@nuxt/test-utils`
    const html = await $fetch('/')

    // 4. Perform checks related to this fixture
    expect(html).toContain('<div>ssr</div>')
  })
})

// 5. Repeat
describe('csr', async () => { /* ... */ })
```

四处细节决定了这段代码能不能跑起来：

- **`setup` 与 `$fetch` 从 `@nuxt/test-utils/e2e` 这个子路径导入**，不是从包根。
- **`await setup(...)` 写在 `describe` 的回调里**，而 `describe` 的回调本身是 `async`。
- **`rootDir` 用的是 `fileURLToPath(new URL('./fixtures/ssr', import.meta.url))`**，也就是把 URL 转成文件系统路径——fixture 目录是相对**测试文件**定位的。
- 第 5 步"重复"的形态就是再开一个 `describe('csr', ...)`：**一个 fixture 对应一组断言，多套配置就多个 describe**。

页面末尾的 `::tip` 给了一个可抄的地方：**这套流程的示例在模块 starter 的 `test/basic.test.ts` 里**。starter 自带 Vitest 与 ESLint，`npm run test` 与 `npm run test:watch` 已经在脚本里了。

## 手工验证：唯一一条走"发布形态"的路

先给 playground 的位置：**开发时有一个 playground 应用来测模块是非常有用的，模块 starter 已经内置了一个**。

但 playground 测的是**仓内源码形态**（`modules: [MyModule]` 那种直连）。要在**你自己的另一个项目**里测（页面原话：不是模块仓库组成部分的那些应用），走这条：

1. 用 `npm pack`（或你的包管理器等价命令）从模块**生成一个 tarball**。
2. 在测试项目里把它加进 `package.json` 的依赖：**`"my-module": "file:/path/to/tarball.tgz"`**。
3. 之后就能像在任何普通项目里那样引用 `my-module`。

这条路的价值在第 1 步：**`npm pack` 出的是准备发到 npm 的那份产物**，装它等于装发布版。前面讲过发布模块时构建会发生两次，而 `file:` 指向 tarball 的做法能把"产物里到底有没有那个文件"这件事在发出之前验掉。**playground 不会告诉你这个**，因为它跑的是源码。

## 三种测法各自能发现什么

| 测法 | 跑的形态 | 能发现 | 发现不了 |
| --- | --- | --- | --- |
| playground | 仓内源码 ＋ 自动导入 | 注入有没有生效、页面渲染对不对 | 发布产物缺文件、`node_modules` 里自动导入失效 |
| E2E fixture | 仓内源码（`modules: [MyModule]`） | 各种配置组合下的构建与渲染结果 | 同上，因为也是源码 |
| `npm pack` ＋ `file:` | 已构建的分发产物 | 入口路径、`exports`、runtime 文件是否随包发出 | 细粒度的逻辑断言（它不是自动化测试） |

`node_modules` 里那件事值得回看：模块结构页警告过，**发布出去的模块在自己的 runtime 目录里用不了自动导入**。这一条**只有在第三种测法里会暴露**，前两种都跑在源码位置，自动导入照常工作。

这一节没写的：fixture 要不要先跑 `nuxt prepare`、多个 fixture 之间是否并行、E2E 与 Vitest 配置文件怎么对应、以及单元/集成测试的官方方案（明确未定）。

上手顺序按这个来：**先给 playground 里手动过一遍，再把每种配置写成 fixture 进 E2E，发布前用 `npm pack` 在别的项目里装一次。**

下一篇是官方给模块作者的九条最佳实践。

## 下一篇

下一篇 640：模块最佳实践九条：1 秒警告、三类前缀表与版本无关的写法

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/testing
