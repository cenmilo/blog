---
url: /posts/d8da6695c57594f89f144259005535c9/
title: 环境覆盖后缀：production、development 与 env
date: 2027-02-04T10:00:00+08:00
updated: 2027-02-04T10:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt.config 允许按环境写不同的配置块，用三个带 $ 前缀的键：$production 与 $development 对应两个内置环境，$env.<名字> 对应你自定义的环境（如 $env.staging）。选环境靠 `nuxt build --envName staging` 这样的 --envName 标志。这套机制底层是 c12 的 environment-specific configuration。这一篇给出三者的官方示例形状、--envName 用法，以及一条给图层作者的补充——用 $meta 键给图层提供元数据。

categories:
  - 前端开发

tags:
  - nuxt4
  - 环境覆盖
  - c12
  - envName
  - nuxt.config
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（345）讲配置分文件层与环境变量层。这一篇是第三种：**在同一个 nuxt.config 里，按环境写不同的块**。官方 configuration 页 "Environment Overrides" 给的就是这个。

## 三个 $ 键

官方定位：

> You can configure fully typed, per-environment overrides in your nuxt.config

例子（官方原样）：

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

三个键各司其职：

- **`$production`**：生产环境下这些配置生效（示例给所有路径开 isr）。
- **`$development`**：开发环境下生效。
- **`$env.<名字>`**：自定义环境，比如 `$env.staging`。$env 是一个对象，你可以往里放任意多个环境名。

这些是**完全类型化**的覆盖（官方说 fully typed），且和顶层配置合并——顶层是公共部分，$production 等是"该环境额外叠加/覆盖"的部分。

## 怎么选环境：--envName

内置的 production/development 跟着构建/开发模式走。自定义环境要显式选，官方：

> To select an environment when running a Nuxt CLI command, simply pass the name to the `--envName` flag, like so: `nuxt build --envName staging`.

也就是 `nuxt build --envName staging` 会激活 `$env.staging` 那块。

## 底层是谁：c12

configuration 页点了机制来源：

> To learn more about the mechanism behind these overrides, please refer to the `c12` documentation on environment-specific configuration.

这套按环境合并配置的能力来自 c12（也是 335 里 .env 支持的底层库）。所以 `$production`/`$env` 的行为语义以 c12 的 environment-specific configuration 为准。

## 图层作者补一句：$meta

configuration 页有条 note 是给图层作者的：

> If you're authoring layers, you can also use the `$meta` key to provide metadata that you or the consumers of your layer might use.

`$meta` 不是环境覆盖、是"给图层挂元数据"，供图层作者或消费方读取。放这里是因为它同属这几个 `$` 前缀键一族，别和 `$production` 混——它不控制环境。

## 自查

- 按环境写不同配置用 `$production` / `$development` / `$env.<名字>`；自定义环境用 `--envName <名字>` 选。
- 顶层配置是公共、这些是环境叠加；机制来自 c12。
- `$meta` 是给图层提供元数据的键，不是环境开关。
- 这和"环境变量覆盖 runtimeConfig"（342/345）是两码事：$production 改的是配置块、NUXT_ 变量改的是 runtimeConfig 的值。

## 下一篇

下一篇 347 讲外部配置文件：nitro.config.ts、vite.config.ts、postcss.config.js、webpack.config.ts 这几个 Nuxt 不再单独读，要写进 nuxt.config 对应键。

## 本文依据

- Configuration（`getting-started/configuration`）："Environment Overrides" 节（"configure fully typed, per-environment overrides in your nuxt.config"、`$production`/`$development`/`$env.staging` 示例、`nuxt build --envName staging`、机制指向 c12 的 environment-specific configuration、note 图层 `$meta` 提供元数据）。
