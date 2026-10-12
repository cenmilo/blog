---
url: /posts/a74a23882da963bfbe78dcea618c17b2/
title: 给组合式函数写测试：mockNuxtImport 与上下文
date: 2027-03-27T09:00:00+08:00
updated: 2027-03-27T09:00:00+08:00
author: cmdragon

summary:
  摘要：给一个用了 `useState`、`useRoute` 的组合式函数写测试，第一件事不是断言，是这些自动导入的函数从哪来。官方的办法是 `mockNuxtImport`——直接替换 Nuxt 的自动导入，三种写法（字符串名、带泛型的字符串名、直接传目标函数），后两种能拿到 `original` 原实现做包装。它有一个硬限制：**同一个测试文件里，对同一个导入只能用一次**，因为它是个被转成 `vi.mock` 的宏，而 `vi.mock` 会被提升。要在不同测试里给不同实现，官方给的是 `vi.hoisted` 先造好 mock 再引用，或者 `mockNuxtImport(useRoute, original => vi.fn(original))` 配 `beforeEach` 里的 `vi.resetAllMocks()`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 组合式函数
  - mock
  - vitest
  - 测试
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

你给一个组合式函数写测试，它里面调了 `useState('cart')`。跑起来要么报 "nuxt instance unavailable"，要么静默返回一个你没法控制的东西。

问题是两层：这个函数需要一个 Nuxt 上下文；而你不想让测试真的去读那个状态。官方的解法也是两层，这一篇讲第二层——把自动导入换掉。

## 前提：上下文由环境给，不由你给

先说清第一层，因为它决定了测试该放哪。文档在 Running Tests 那条 `::warning` 里写了环境事实：

> When you run your tests within the Nuxt environment, they will be running in a happy-dom or jsdom environment. Before your tests run, a global Nuxt app will be initialized (including, for example, running any plugins or code you've defined in your `app.vue`).

也就是说：在 Nuxt 环境里跑测试，**框架已经替你建了一个全局 Nuxt 应用**，插件会执行，`app.vue` 里的代码也会执行。上下文来自这里，不需要你自己 `createNuxtApp`。

同一条警告的后半句是这次的主角：**要特别小心别改动全局状态，需要改的话就在之后把它重置**。你的测试跑在同一个全局应用上，前一个测试改的东西后一个还看得见。

另一条边界在 Organizing Your Tests 的 `::important` 里，是给你的组合式函数测试选址用的：**单元测试不应该依赖 Nuxt 运行时特性，比如自动导入或组合式函数**。所以凡是里面用了 `useState`/`useRoute` 的，就得放进 Nuxt 那个 project（推荐目录 `test/nuxt/`），不是 `test/unit/`。

## mockNuxtImport 的三种写法

定义一句：**"allows you to mock Nuxt's auto import functionality."** 最朴素的形态是替换 `useState`：

```ts
import { mockNuxtImport } from '@nuxt/test-utils/runtime'

mockNuxtImport('useState', () => {
  return () => {
    return { value: 'mocked storage' }
  }
})

// your tests here
```

第一个参数是自动导入的名字（字符串），第二个是工厂函数。官方文档接着给了两种进阶写法，说明是"为了类型安全可以显式给 mock 标类型，并且可以用工厂函数收到的原始实现来 mock 更复杂的功能"：

```ts [test/nuxt/import.test.ts]
import { mockNuxtImport } from '@nuxt/test-utils/runtime'

mockNuxtImport<typeof useState>('useState', (original) => {
  return (...args) => {
    return { ...original('some-key'), value: 'mocked state' }
  }
})

// or specify the target to mock
mockNuxtImport(useState, (original) => {
  return (...args) => {
    return { ...original('some-key'), value: 'mocked state' }
  }
})

// your tests here
```

三种写法的差别可以列成表：

| 写法 | 第一参数 | 工厂能否拿到 `original` | 类型检查 |
| --- | --- | --- | --- |
| `mockNuxtImport('useState', () => ...)` | 字符串名 | 示例里没有第二参数 | 无显式类型参数 |
| `mockNuxtImport<typeof useState>('useState', (original) => ...)` | 字符串名 + 泛型 | 能 | 显式指定被 mock 函数的类型 |
| `mockNuxtImport(useState, (original) => ...)` | 直接传函数本身（注释写作 "specify the target to mock"） | 能 | 由传入的目标推断 |

第三种写法里 `useState` 是自动导入进来的符号，示例文件顶部并没有 import 它——官方示例文件放在 `test/nuxt/`，正是上面那条 TypeScript 上下文规则覆盖的目录。

## 硬限制：一个文件里同一个导入只能一次

这条写成 `::note`，是这一节最容易撞的：

> `mockNuxtImport` can only be used once per mocked import per test file. It is actually a macro that gets transformed to `vi.mock` and `vi.mock` is hoisted, as described in the Vitest docs.

三个信息：限制的范围是**每文件、每个被 mock 的导入**；它的实现身份是**宏**，编译时变成 `vi.mock`；`vi.mock` 会被**提升**到文件顶部，所以你在测试里"按条件再 mock 一次"没有意义——两次调用都会落到同一处，后写的不会覆盖前写的执行时机。

## 两种绕法

**要在不同测试里换实现**——官方给的方案是用 `vi.hoisted` 先把 mock 造出来并暴露，再在 `mockNuxtImport` 里引用它：

```ts
import { vi } from 'vitest'
import { mockNuxtImport } from '@nuxt/test-utils/runtime'

const { useStateMock } = vi.hoisted(() => {
  return {
    useStateMock: vi.fn(() => {
      return { value: 'mocked storage' }
    }),
  }
})

mockNuxtImport('useState', () => {
  return useStateMock
})

// Then, inside a test
useStateMock.mockImplementation(() => {
  return { value: 'something else' }
})
```

`vi.hoisted` 的作用就是让这段创建代码跟着被提升，于是顶层的 `mockNuxtImport` 工厂能安全引用 `useStateMock`。文档在这段后面加了一句提醒：注意在每次测试前后**恢复 mock**（链接指向 Vitest 的 `mockrestore`），否则测试之间的 mock 状态会串。

**只想在某个测试里改行为**——用 `vi.fn(original)` 包一层，配合 `resetAllMocks`：

```ts
import { beforeEach, vi } from 'vitest'
import { mockNuxtImport } from '@nuxt/test-utils/runtime'

mockNuxtImport(useRoute, original => vi.fn(original))

beforeEach(() => {
  vi.resetAllMocks()
})

// Then, inside a test
const useRouteOriginal = vi.mocked(useRoute).getMockImplementation()!
vi.mocked(useRoute).mockImplementation(
  (...args) => ({ ...useRouteOriginal(...args), path: '/mocked' }),
)
```

这段的模式值得记住三步：包一层可记录的 `vi.fn`、每个测试前重置、测试内先用 `getMockImplementation()` 把原实现取出来，再在 `mockImplementation` 里展开它并只改你要改的那个字段（示例里是 `path: '/mocked'`）。这样 `useRoute` 的其余返回值保持真实，不会因为 mock 而丢字段。

## 自查

- 上下文不用自己建：在 Nuxt 环境跑测试时全局 Nuxt 应用已初始化，插件与 `app.vue` 代码会执行；代价是要小心全局状态，用完重置。
- 用了自动导入或组合式函数的测试不属于 `test/unit/`，`::important` 明确说单元测试不该依赖这些。
- `mockNuxtImport` 三种写法：字符串名、字符串名 + `<typeof useState>`、直接传目标函数；后两种工厂能拿到 `original` 做包装。
- 一个文件对同一个导入只能调用一次；它是转成 `vi.mock` 的宏，会被提升。
- 换实现用 `vi.hoisted` 暴露 mock 再引用，记得恢复；测试内临时改行为用 `vi.fn(original)` + `resetAllMocks` + `getMockImplementation()` 展开原值。

## 下一篇

下一篇 499 讲服务端路由的测试：e2e 那侧怎么真的打到 `/api/...`，以及单元侧用 `registerEndpoint` 造假接口那条路。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/testing（Helpers 一节 `mockNuxtImport`：定义句、`useState` 最小示例、带泛型与直接传目标两种写法及 "specify the target to mock" 注释、"can only be used once per mocked import per test file" 的 `::note`、`vi.hoisted` 方案与恢复 mock 的提醒、`mockNuxtImport(useRoute, original => vi.fn(original))` 配 `beforeEach`/`getMockImplementation` 的示例）
- https://nuxt.com/docs/4.x/getting-started/testing（Running Tests 的 `::warning`：happy-dom 或 jsdom、全局 Nuxt 应用先初始化、勿改动全局状态；Organizing Your Tests 的 `::important`：单元测试不应依赖自动导入与组合式函数；TypeScript 小节 `test/nuxt/` 与 `tests/nuxt/` 的上下文归属）
