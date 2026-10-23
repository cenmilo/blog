---
url: /posts/91e256d9c4f612b72514efa19403d702/
title: 363 - 多环境配置演练：dev、staging、prod 三套
date: 2027-02-10T09:00:00+08:00
updated: 2027-02-10T09:00:00+08:00
author: cmdragon

summary:
  摘要：本地开发想关掉一堆东西、线上要全站缓存、还要加一套预发环境专门验一遍——同一份 `nuxt.config` 不想改来改去。Nuxt 支持在配置里按环境写覆盖块：内置的 `$production`、`$development`，以及自定义的 `$env.staging`；跑命令时用 `--envName staging` 选自定义环境。这套是 `c12` 的环境机制，别和加载 `.env` 文件的 `--dotenv` 搞混——一个选配置块、一个选密钥文件。这一篇把三种覆盖块、两条选环境的路子、以及和 `.env` 的分工讲清，给一份 dev/staging/prod 三套的完整写法。

categories:
  - 前端开发

tags:
  - nuxt4
  - 多环境配置
  - envName
  - nuxt-config
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


一个站往往要跑在三套地方：本地 `nuxt dev`、给测试用的预发（staging）、正式线上（production）。差别常常是几行配置——线上要全站边缘缓存、本地不想缓存；预发要指向测试数据库、线上指向生产库。把这些全写进顶层 `nuxt.config`，就得每次部署前手动改、极易改漏。Nuxt 允许在同一个配置文件里**按环境分块**。

## "环境"在这里指什么

这里的"环境"不是操作系统的环境变量，而是配置加载时挑的那一套覆盖块。Nuxt 的配置合并走的是 `c12`（官方原话："To learn more about the mechanism behind these overrides, please refer to the `c12` documentation on environment-specific configuration."）。它认两类选择器：

- **内置环境名**：`$production`、`$development`，对应构建/运行时的环境，自动挑，不用你传参。
- **自定义环境名**：写在 `$env.<名字>` 下，跑命令时用 `--envName <名字>` 显式选中。

## 三种覆盖块

`getting-started/configuration` 的 Environment Overrides 给的例子，官方原样如下：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  $production: {
    routeRules: {
      '/**': { isr: true },
    },
  },
  $development: {
    //
  },
  $env: {
    staging: {
      //
    },
  },
})
```

三个信息：

1. `$production` 块里配的，只在 production 生效。这里给 `/**` 全站开了 `isr: true`（372 之前讲的 route rules，见 365-367），线上才要、本地不要。
2. `$development` 块给本地开发用（示例留空）。
3. `$env.staging` 是自定义环境块，名字随你起（`staging`、`preview` 都行）。

它开头有一句定性："You can configure **fully typed**, per-environment overrides in your nuxt.config."——这些块是**有类型提示**的，键写错编辑器会报。

## 选中自定义环境：`--envName`

`$production`/`$development` 自动挑，但自定义的 `$env.staging` 不会自己生效，要在命令里点名：

> To select an environment when running a Nuxt CLI command, simply pass the name to the `--envName` flag, like so: `nuxt build --envName staging`.

所以预发构建就是 `nuxt build --envName staging`。没点名的环境块不合并进来。

## 图层里还有个 `$meta`

官方在同一个例子后补了一句：

> If you're authoring layers, you can also use the `$meta` key to provide metadata that you or the consumers of your layer might use.

`$meta` 是给写 layer 的人挂元数据用的，不是环境开关——别把它当第四个环境块。

## 别和 `--dotenv` 混：那是选密钥文件

多环境经常还要配一套不同的密钥（预发一套、生产一套）。**选配置块**和**选 `.env` 文件**是两条独立的路子，很容易搅在一起：

- `--envName staging`：挑 `nuxt.config` 里哪个 `$env` 块。
- `--dotenv .env.local`：加载哪个环境变量文件。`.env` 那页给的用法：

```bash [Terminal]
npx nuxt dev --dotenv .env.local
```

而且 `.env` 只在 dev、build、generate 三个时机被读（"it will be automatically loaded **at dev, build and generate time**"），产物跑起来时不再读——这条边界 372 前详讲，部署侧的环境变量得交给平台设。

## 三套一起写的完整形态

把上面拼起来，一份能同时喂 dev / staging / prod 的配置大概长这样（配置块的键用文档给的，密钥走 runtimeConfig + 环境变量）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  // 三套共用的部分放顶层
  runtimeConfig: {
    apiSecret: '',           // 私有键，运行时靠 NUXT_API_SECRET 覆盖
    public: { apiBase: '/api' },
  },

  // 本地：不缓存、多开点调试
  $development: {
    // debug: true 之类放这
  },

  // 预发：构建时用 nuxt build --envName staging 选中
  $env: {
    staging: {
      routeRules: {
        '/**': { swr: 60 },   // 短缓存，方便反复验
      },
    },
  },

  // 线上：自动生效，全站边缘缓存
  $production: {
    routeRules: {
      '/**': { isr: true },
    },
  },
})
```

要点：顶层写"三套都要"的；`$development`/`$production` 各写只属于那一套的；`$env.staging` 写预发独有、且必须 `--envName staging` 才生效。密钥不进这些块、进 `.env` 或部署平台的环境变量，靠对得上的 `NUXT_` 变量运行时覆盖（见 342、361）。

## 自查

- 内置 `$production`/`$development` 自动按环境挑；自定义 `$env.<名>` 要 `nuxt build --envName <名>` 才生效。
- 覆盖块是 fully typed 的，键写错有提示。
- `$meta` 是图层元数据，不是环境块。
- `--envName`（选配置块）与 `--dotenv`（选 .env 文件）是两回事，别混。
- 共用配置放顶层，各环境差异放各自块；密钥走环境变量不走配置块。

## 下一篇

下一篇 364 收 340-364 那几篇：一份配置排错清单，值没生效时按顺序检查哪几处（键写错、优先级、外部文件、runtimeConfig 映射、生效时机、`--envName` 没选）。
