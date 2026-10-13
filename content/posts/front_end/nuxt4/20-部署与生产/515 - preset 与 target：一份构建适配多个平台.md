---
url: /posts/668d9f2dbd15d4f55e9340725dbee726/
title: 515 - preset 与 target：一份构建适配多个平台
date: 2027-04-01T11:00:00+08:00
updated: 2027-04-01T11:00:00+08:00
author: cmdragon

summary:
  摘要：决定产物形状的是 preset，写入位置有三处：配置里的 `nitro.preset`、环境变量 `NITRO_PRESET`（其次是 `SERVER_PRESET`）、命令行的 `--target`（`--preset` 只是它的旧名）；`--prerender` 会把 target 强制成 `static`。**4.x 的 Nuxt 配置参考里几乎没有 preset 这个键**——它是 Nitro 的配置项、经 `nitro` 节透传，完整清单在 `nitro.build/deploy`；官方把这种透传标为高级选项，警告 Nitro 的配置接口可能随 Nuxt 的 semver-minor 版本变化并影响生产部署。另外两件事与平台有关：Netlify 与 Vercel 会把路由规则尽量映射到平台原生规则；有些预设的产物不能在本地起服务器，`nuxt preview` 会改跑该预设自己的预览命令并打印它在跑什么。

categories:
  - 前端开发

tags:
  - nuxt4
  - 部署
  - preset
  - nitro
  - 构建
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

同一份代码，今天要在自己的机器上跑，明天可能要传到 Vercel。代码不用动，动的是**构建出什么形状**——这件事在 Nuxt 里由一个值决定：preset。

## 三个写入位置

部署页的 Presets 一节给的定性：**"In addition to Node.js servers and static hosting services, a Nuxt project can be deployed with several well-tested presets and a minimal amount of configuration."**

写法两种，官方都给了代码。写在配置里：

```ts twoslash [nuxt.config.ts]
// @errors: 2353
export default defineNuxtConfig({
  nitro: {
    preset: 'node-server',
  },
})
```

或用环境变量跑构建：

```bash [Terminal]
NITRO_PRESET=node-server nuxt build
```

第三种在命令行上：`nuxt build` 与 `nuxt generate` 都接受 `--target=<target>`，官方说明是 **"Deploy target for the configured server builder (e.g. `node-server`, `vercel`, `netlify`)"**。

三者谁优先，build 命令页那句话是全的：

> `--preset` is still accepted as another name for `--target`. When neither is passed, the `NITRO_PRESET` and `SERVER_PRESET` environment variables are used, in that order.

拆成顺序表：

| 你给了什么 | 生效的是 |
| --- | --- |
| `--target=x` | `x` |
| `--preset=x`（旧名） | 等同于 `--target=x` |
| 两个都不给，环境里有 `NITRO_PRESET` | `NITRO_PRESET` 的值 |
| 只有 `SERVER_PRESET` | `SERVER_PRESET` 的值（顺序在后） |
| 命令行与环境变量都没有 | 走配置里的 `nitro.preset`；都没有则按自动识别，识别不到时默认 node-server |

最后一行的前半句是 `nitro: { preset: ... }` 那段示例支持的，后半句来自部署页那句 "Default output format if none is specified or auto-detected"。**"配置文件与自动识别之间的确切优先级，文档没有写成一张表"**，要确认就在构建日志里看它按哪个预设打包。

有一条会覆盖你的选择：`--prerender` **总是**把 target 设成 `static`（build 命令页的 `::note` 原句是 "`--prerender` will always set the target to `static`"）。所以同时传 `--prerender` 与 `--target=vercel` 时，别指望 target 还留着。

## 4.x 的配置参考里几乎没有 preset

这是最让人找不到东西的地方。`api/nuxt-config` 那条配置条目对 `nitro` 这个顶层节的记载只有两句：**"Configuration for Nitro."** 与一个指向 **Nitro configuration docs** 的链接，展开的子项只有 `routeRules`（类型 `object`）与 `runtimeConfig`（类型 `object`，默认值那份 JSON 里是 `public: {}`、`app` 下四个键、`nitro.envPrefix: "NUXT_"`）。

也就是说：**`preset` 这个键不在 Nuxt 的配置参考里列着，它是 Nitro 的配置项，经 `nitro` 节透传。** 完整清单在 Nitro 那边——部署页那句 "Check the Nitro deployment for all possible deployment presets and providers." 指向 `nitro.build/deploy`。

透传这件事官方还专门警告过一次。`directory-structure/server` 的 Nitro Config 小节：**"You can use `nitro` key in `nuxt.config` to directly set Nitro configuration."**，紧跟的 `::warning` 是：

> This is an advanced option. Custom config can affect production deployments, as the configuration interface might change over time when Nitro is upgraded in semver-minor versions of Nuxt.

这条警告的读者恰好是把 Nitro 配置写进 `nuxt.config` 的人：**Nuxt 的 semver-minor 升级就可能换掉 Nitro 的配置接口，而你的自定义配置会影响生产部署。** 所以只把必需项写进 `nitro` 节，别整份抄 Nitro 示例。

## preset 会反过来影响你能怎么测它

两个方向都有记载。

**部署平台的原生规则。** 混合渲染那段有一句：**"Whenever possible, route rules will be automatically applied to the deployment platform's native rules for optimal performances (Netlify and Vercel are currently supported)."** 也就是说选了这两个平台的预设，你的路由规则会尽量翻译给平台自己的机制，而不是只在 Nuxt 服务器里执行。

**预览这一环会跟着变。** `nuxt preview` 那一节写明：**"Some Nitro presets do not produce a server that can be run locally. For those, the preset's own preview command is run instead, and the command tells you what it is running."** 换个角度读这句：**本地预览能不能起来，是预设的属性，不是你的配置错了**。而命令会打印它到底在跑什么，这是官方唯一让你从输出里读出"这份产物按哪种方式起"的地方。

还有一类是进程模型的预设：`NITRO_PRESET=node_cluster` 用来用 Node 的 cluster 模块做多进程，官方说负载默认按 round robin 分给 worker。它在部署形态上仍属 Node 托管，差别只在起几个进程。

## 想知道自己的项目实际解析到什么

`nuxt info` 那条命令页写明它报**解析到的** Nuxt、Nitro、Vite 版本，以及包管理器、加载的模块与 build modules。版本对不上时，先跑它，再判断是不是 Nitro 版本导致某个预设或配置项不存在。

## 自查

- 三种写法：`nitro.preset`（配置）、`NITRO_PRESET`/`SERVER_PRESET`（环境变量，按这个顺序）、`--target`（`--preset` 是旧名）。
- `--prerender` 会把 target 强制成 `static`，与 `--target` 同时给时不要期待 target 保留。
- **`preset` 不在 Nuxt 的配置参考里**——它是 Nitro 的配置项经 `nitro` 节透传，完整清单在 `nitro.build/deploy`。
- 往 `nitro` 节写自定义配置是官方标注的高级选项，Nitro 配置接口可能随 Nuxt 的 semver-minor 变化，会影响生产部署。
- Netlify 与 Vercel 会把路由规则尽量映射到平台原生规则；有些预设本地产物不可本地起服务器，`nuxt preview` 会改跑预设自己的命令并打印它在跑什么。

## 下一篇

下一篇 516 讲子路径部署：`app.baseURL` 与 `NUXT_APP_BASE_URL`，以及那条硬限制——`nuxt.config.ts` 里**不支持 `./` 这类相对路径**（原因是 Nitro 的限制），官方给了两种替代写法。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/deployment（Presets 一节的定性句、`nitro: { preset: 'node-server' }` 示例、`NITRO_PRESET=node-server nuxt build`、指向 nitro.build/deploy 的清单链接；Node server preset 的 "Default output format if none is specified or auto-detected"；Cluster Mode 的 `node_cluster` 与 round robin）
- https://nuxt.com/docs/4.x/api/commands/build（`--target` 说明与三个示例值、`--preset` 是别名与 `NITRO_PRESET`→`SERVER_PRESET` 顺序、`--prerender` 总是设 target 为 static 的 `::note`）
- https://nuxt.com/docs/4.x/api/commands/generate（同样带 `--target` 选项）
- https://nuxt.com/docs/4.x/api/nuxt-config（`## nitro` 整节只有两句与两个子项、`runtimeConfig` 默认 JSON 含 `nitro.envPrefix: "NUXT_"`）
- https://nuxt.com/docs/4.x/directory-structure/server（Nitro Config 小节与会影响生产部署的 `::warning`）
- https://nuxt.com/docs/4.x/guide/concepts/rendering（路由规则尽量映射到平台原生规则、当前支持 Netlify 与 Vercel）
- https://nuxt.com/docs/4.x/api/commands/preview（不可本地运行的预设改跑预设自己的 preview 命令并告知）
