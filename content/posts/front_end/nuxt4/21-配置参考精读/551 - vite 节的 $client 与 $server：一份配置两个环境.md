---
url: /posts/d19da23a7dc955e40303fcde79619681/
title: 551 - vite 节的 $client 与 $server：一份配置两个环境
date: 2027-04-13T11:00:00+08:00
updated: 2027-04-13T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`vite` 节第一句是**"Configuration that will be passed directly to Vite."**，第二句给出这一节最重要的规则：**"Top-level `vite` options are shared across both client and server environments. Use `$client` and `$server` to provide environment-specific configuration that will be merged into their respective builds."**——顶层 `vite` 的选项**客户端与服务端共用**，要分开配就写 `$client`（"merged into Vite's configuration for the client (browser) build"）与 `$server`（服务端那次构建），两项类型都是 `object`、都没给默认值。节末还有一句必须读的限定：**"Please note that not all vite options are supported in Nuxt."**（并非所有 Vite 选项在 Nuxt 里受支持）。官方示例给的是 `$client` 里做 `manualChunks`、`$server` 里设 `sourcemap: 'inline'`。

categories:
  - 前端开发

tags:
  - nuxt4
  - vite
  - 构建
  - 客户端服务端
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

同一条 `vite.build.rollupOptions` 改完，浏览器产物按你想的变了，服务端产物也跟着变——而你可能只想要前者。

这就是 `vite` 这一节要解决的分层问题。

## 两句定性与一句限定

`## vite` 开头三句：

> Configuration that will be passed directly to Vite.
>
> Top-level `vite` options are shared across both client and server environments. Use `$client` and `$server` to provide environment-specific configuration that will be merged into their respective builds.

然后是节末的 See 与那句限定：

> **See**: Vite configuration docs for more information.
> Please note that not all vite options are supported in Nuxt.

三件事按重要性排：

1. **这份配置是"直接传给 Vite"**——它的字段形状跟随 Vite 自己的配置，Nuxt 不重新定义。
2. **顶层选项是两边共用的**。写在 `vite` 第一层的每一项，客户端与服务端两次构建都会读到。
3. **"not all vite options are supported"**——能传不等于生效。官方把这句放在 See 后面，没给"哪些不受支持"的清单。

## `$client` 与 `$server` 各管一次构建

两个子项的说明句只差一个词：

| 子项 | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `$client` | Configuration that will be merged into Vite's configuration for the **client (browser) build**. | `object` | 未列 |
| `$server` | Configuration that will be merged into Vite's configuration for the **server build**. | `object` | 未列 |

用词是 **merged into**（并入各自那次构建的配置），不是"替换整份 Vite 配置"。所以形状是：**顶层 `vite` 给公共部分，`$client`/`$server` 再往对应那一侧并进去。**

官方示例正好演了这种分工：

```ts
export default defineNuxtConfig({
  vite: {
    $client: {
      build: {
        rollupOptions: {
          output: {
            manualChunks: {
              analytics: ['analytics-package'],
            },
          },
        },
      },
    },
    $server: {
      build: {
        sourcemap: 'inline',
      },
    },
  },
})
```

两侧各自做的事很清楚：**`$client` 里做代码分块**（`manualChunks` 把某个包切成一块，这是浏览器产物的事），**`$server` 里把 sourcemap 设成 inline**（服务端产物读栈时不必再找 `.map` 文件）。

一个对照值得记住：`sourcemap` 那件事在 Nuxt 也有自己的开关——配置参考里有 `## sourcemap` 一节，官方在调试页写的是"服务端构建默认开、客户端只在 dev 开"。**`vite.$server.build.sourcemap` 是 Vite 层的写法，`sourcemap` 是 Nuxt 层的开关，两处都能影响服务端产物，官方没写它们之间的优先关系。**

## 什么时候必须分环境

从示例给的两个用途能推出判断线：**只在浏览器侧有意义的构建策略**（分块、预加载、资源引用处理）写在 `$client`；**只在服务端侧有意义或形态不同**的（inline sourcemap、外部依赖处理）写在 `$server`。

反过来，两边行为本来就一致的（比如一个 `plugins` 项、一个 `define` 常量），写在顶层一次就够，**不需要复制两份**。

要确认自己写对了没有，最直接的动作是两次构建各看一次产物：`npx nuxt build` 完成后检查 `.output` 里公共 chunk 与服务端入口 `.output/server/index.mjs` 的形状是否符合预期，而不是只看浏览器侧。

## 一处命名细节

`$client` 与 `$server` 是**以 `$` 开头的键名**。这与 Nuxt 里另一个 `$` 前缀的用法（`$fetch` 那个全局取数工具）没有关系，也不同于环境覆盖那类 `$production`／`$development`／`$env` 的写法——三种 `$` 出现在不同层，别互相搬。

**文档没有解释为什么这两个键要带 `$`**，能确证的只有它们的官方定义句与"merged into their respective builds"这个行为描述。

## 自查

- `vite` 是"直接传给 Vite"的配置，**顶层选项客户端与服务端共用**。
- 分环境要写 `$client`／`$server`（都 `object`、都没有默认值），官方措辞是 **merged into** 各自那次构建的配置。
- 示例的分工：`$client` 里 `build.rollupOptions.output.manualChunks`，`$server` 里 `build.sourcemap: 'inline'`。
- 官方明写**并非所有 Vite 选项在 Nuxt 里受支持**，且没有给不受支持清单——加了没效果要回来看这句。
- `vite.$server.build.sourcemap` 与 Nuxt 的 `sourcemap` 开关分属两层，两者优先关系文档未写。

## 下一篇

下一篇 552 读 `vite` 节的其他子项：`cacheDir`、`clearScreen`、`define` 那五个默认常量、`esbuild` 的四项默认、`mode` 默认 `"production"`、`optimizeDeps.exclude` 里的 `vue-demi`、`resolve.extensions` 与 `root`，以及 `server.fs.allow` 那份四行默认清单。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## vite` 整节：pass directly to Vite 与 top-level shared／`$client`／`$server` merged into respective builds 两句、`### $client` 与 `### $server` 的说明句和 `object` 类型（均无默认）、那份 `$client.manualChunks` 与 `$server.sourcemap` 示例全文、See 指向 Vite 配置文档与"not all vite options are supported in Nuxt"一句；对照的 `## sourcemap` 节）
- https://nuxt.com/docs/4.x/guide/going-further/debugging（source map 默认"服务端开、客户端仅 dev"，用于说明两层开关并存）
