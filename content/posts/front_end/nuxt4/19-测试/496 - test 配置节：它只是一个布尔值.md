---
url: /posts/b07bf0eb8514490f5d017a1bf575def4/
title: test 配置节：它只是一个布尔值
date: 2027-03-26T10:00:00+08:00
updated: 2027-03-26T10:00:00+08:00
author: cmdragon

summary:
  摘要：`nuxt.config.ts` 里确实有一个 `test` 节，但它的完整定义只有三行：说明是 "Whether your app is being unit tested."，类型 `boolean`，默认 `false`。它不是测试配置的集合处——Vitest 的选项写在 `vitest.config.ts`（那里也有一个 `test` 字段，两者毫无关系），测试文件的目录叫 `test/`，`nuxt test` 那条命令设置的是 `process.env.NODE_ENV`。这一篇把这几个同名但不同层的东西分开，并说清这个布尔值目前能确证到什么程度。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt-config
  - 测试
  - vitest
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

你想给测试留点配置。打开 `nuxt.config.ts`，在按字母排的顶层节里翻到 `test`——它在那儿，夹在 `theme` 前面。

问题来了：`test` 里能写什么？

## 它的完整定义就是这三行

`api/nuxt-config` 那一页对 `test` 的记载，一字不多：

> Whether your app is being unit tested.

- **类型**：`boolean`
- **默认**：`false`

配置文档那一页是 60 个顶层节的合集，每个节的写法都是「一句说明 + 类型 + 默认值」，`test` 就是其中最省的一节。**它没有子项、没有对象形态、没有列任何可选字段。**

所以"把测试相关配置收进 `nuxt.config` 的 `test` 节"这个想法，按文档是不成立的：这一节只回答一个问题——这个应用是不是正在被单元测试。

## 三个同名不同层的东西

混淆来自名字撞车。四件事各自的归属：

| 你会看到的东西 | 它在哪 | 它管什么 | 文档依据 |
| --- | --- | --- | --- |
| `test`（`boolean`） | `nuxt.config.ts` 顶层节 | 应用是否正在被单元测试，默认 `false` | `api/nuxt-config` |
| `test`（对象） | `vitest.config.ts` | Vitest 自己的配置字段，例如 `projects`、`globals` | 测试页给的三份配置示例 |
| `test/` | 项目根目录 | 放测试文件的推荐目录，Nuxt 不扫它 | `directory-structure/test` |
| `nuxt test` | CLI 命令 | 用 `@nuxt/test-utils` 跑测试，顺带在 `NODE_ENV` 未设时设成 `test` | `api/commands/test` |

第二行最容易出事。测试文档里那两份配置长这样：

```ts
  test: {
    // Vitest 的字段
  },
```

它出现在 `vitest.config.ts`（以及 `defineVitestProject` 返回的那份配置）里，是 Vitest 的字段名。把它挪进 `nuxt.config.ts`，你写下的就是一个 `boolean` 节的非法值。

反过来也一样：`@nuxt/test-utils/runtime` 那套 mock 的开关不写在 `nuxt.config.ts` 的 `test` 里，而是写在 Vitest project 的 `environmentOptions.nuxt.mock` 下，比如 `intersectionObserver` 默认 `true`、`indexedDB` 默认 `false`（配置键写作 `indexedDb`）。

## 谁把它设成 true：文档没写

能确证的只有三条：

- 它默认 `false`，说明常态下没有被打开。
- `nuxt test` 那条命令页只提 `process.env.NODE_ENV`，**没有一个字提到 `test` 配置节**。
- 整份 4.x 文档里，我没有找到读取这个节的示例或 API 说明——模块钩子表、Nuxt Kit 函数列表里都没有以它为依据的条目。

也就是说：**「框架在什么时机把它置为 `true`」这一条，文档未说明**。你要是想靠它做分支（比如测试时不注册某个插件），别按推测写，先自己取证。

验证方法很短。在 `nuxt.config.ts` 里临时打印：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  test: false,
  hooks: {
    'modules:before'() {
      console.log('[probe] config.test =', JSON.stringify(this.test))
    },
  },
})
```

然后分别跑 `npx nuxt dev`、`npx nuxt build`、`npx vitest --project nuxt`，看那一行输出各自是什么。这里的 `this.test` 形态如果与你的 Nuxt 版本不符，直接用 `npx nuxt analyze` 之前先跑一次 `npx nuxt prepare`，看生成的 `.nuxt` 类型里 `test` 的声明——文档说 `nuxt prepare` 用于生成类型，这是能读到配置节实际形态的地方。

（上面这段是取证的思路，不是官方示例；官方在配置文档里对这一节没有给任何示例代码。）

## 真正能写的替代方案

如果你的目的是"测试时让配置不一样"，文档里有记载的是另一条路：**环境覆盖**。`nuxt.config.ts` 支持 `$production`、`$development`、`$env: { staging: {...} }` 这类节，用 `nuxt build --envName staging` 指定环境。测试文档里那句 Note 正是走这条路：组件请求外部 API 时，给 `baseURL`，再通过环境覆盖配置（那里写作 `$test`）把它置空，于是所有请求都落到 Nitro 服务器上——也就是落到你 `registerEndpoint` 造的假接口。

**注意 `$test` 这一节只出现在测试页那句 Note 里**，配置文档列举环境覆盖时给的是 `$production`、`$development`、`$env`，没有把 `$test` 列进去。要用它，先按上一段的方法确认你的版本认这个键。

## 自查

- `test` 是 `boolean`、默认 `false`，整节定义只有 "Whether your app is being unit tested." 一句，无子项。
- `vitest.config.ts` 里的 `test` 是 Vitest 的字段，两者同名不同层，别互相搬。
- 文档没有说明谁把它置为 `true`，也没有读取它的 API 示例；要用它做分支，先在本地取证。
- 想按测试环境改配置，有据可依的是环境覆盖节（`$production`/`$development`/`$env`），`$test` 只在测试页一句 Note 里出现。
- mock 开关的位置在 Vitest project 的 `environmentOptions.nuxt.mock`，不在这个节里。

## 下一篇

下一篇 497 讲 `test/` 目录：官方对它的定性是「Nuxt 不会像扫 `app/` 和 `server/` 那样扫它」，以及三个推荐子目录各自对着哪种运行环境。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## test` 一节全部记载：说明句、Type `boolean`、Default `false`；顶层节按字母排列、`theme` 一节相邻）
- https://nuxt.com/docs/4.x/api/commands/test（该页只记载 `NODE_ENV` 设为 `test`，未提及 `test` 配置节）
- https://nuxt.com/docs/4.x/getting-started/testing（Vitest 配置示例中的 `test` 字段、`environmentOptions.nuxt.mock` 的 `intersectionObserver`/`indexedDb` 默认值、`registerEndpoint` 那节指向环境覆盖 `$test` 的 Note）
- https://nuxt.com/docs/4.x/getting-started/configuration（Environment Overrides 小节：`$production`/`$development`/`$env.staging` 与 `--envName`）
- https://nuxt.com/docs/4.x/directory-structure/test（`test/` 目录的推荐用途）
