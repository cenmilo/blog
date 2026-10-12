---
url: /posts/8e3139986ad354ef8a64c50eb41c5ac6/
title: 构建与模块的十个实验开关：watcher 的四种取值与 buildCache 的九个文件
date: 2027-06-04T09:00:00+08:00
updated: 2027-06-04T09:00:00+08:00
author: cmdragon

summary:
  摘要：`watcher` 有四种取值，默认是 `chokidar-granular`（会忽略 `node_modules`、`.git` 这类顶层目录），改成 `parcel` 用 `@parcel/watcher`，文档说这**在大型项目或 Windows 平台上可能改善性能**；`'builder'` 是复用构建器自己的监听器，webpack 与 rspack 没实现自己的 watcher，此时 Nuxt 会打一条警告并退回默认选择。`buildCache` 默认关，开了以后有九个文件一动就触发全量重建，而且**最多只保留 10 份缓存 tarball**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 实验特性
  - 构建缓存
  - 文件监听
  - vite
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## `watcher`：四种取值，各有一个前提

原文的定位：设置一个替代的 watcher，作为 Nuxt 的监听服务。四种取值文档逐个说明：

| 取值 | 行为 |
| --- | --- |
| `chokidar-granular`（默认） | 忽略被排除在监听之外的顶层目录（例如 `node_modules` 与 `.git`） |
| `parcel` | 使用 `@parcel/watcher`，**原文说在大型项目或 Windows 平台上可能改善性能** |
| `chokidar` | 监听源目录里的**所有**文件 |
| `builder` | 复用当前构建器自己的文件监听器（例如 Vite 的 `server.watcher`），而不是再起一个，减少 dev 模式里活跃的监听器数量 |

`'builder'` 那一档还带两句：设成 `future.compatibilityVersion` 为 `5` 时它成为默认；**如果当前构建器没有实现自己的 watcher（文档写明目前是 webpack 与 rspack），Nuxt 会记录一条警告并退回到它的默认选择**。

写法是字符串值：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    watcher: 'chokidar-granular', // 'chokidar', 'parcel' or 'builder' are also options
  },
})
```

## `buildCache`：省时间的前提是知道哪些文件会绕过它

默认关。开启后按**配置与源文件的哈希**缓存 Nuxt 构建产物，作用范围原文限定得很窄：只对 `srcDir` 与 `serverDir` 里属于 Vue/Nitro 那部分的源文件有效。

两个必须知道的限制：

- 下面九个文件**任何改动都会触发全量重建**：`.nuxtrc`、`.npmrc`、`package.json`、`package-lock.json`、`yarn.lock`、`pnpm-lock.yaml`、`tsconfig.json`、`bun.lock`、`bun.lockb`。
- **Nitro 永远会被重建**。原文的说明是 work in progress——正在做让 Nitro 能宣告它的可缓存产物与哈希。

还有一条 note：**最多保留 10 份缓存 tarball**。

所以它的实际收益主要来自 Vue 的 client/server 包，而改动 `srcDir` 一定重建那两份；配置文件与锁文件一动就没有增量可言。

## `checkOutdatedBuildInterval`：它是 `appManifest` 的下游

这一条管"检查新构建的时间间隔（毫秒）"，示例给的是 `3600000, // 1 hour, or false to disable`。**当 `experimental.appManifest` 为 `false` 时它整个被禁用**，设成 `false` 也是关掉。文档没有列出它的默认数值——示例里那个 1 小时是示例，不是默认值声明。

## `appManifest` 与 `noVueServer`：两个"关掉服务端某块能力"的开关

`appManifest` 原文一句：**用 app manifest 在客户端侧遵守路由规则**。默认开，可以关。它同时是上面那条检查间隔的前提。

`noVueServer` 则是**禁用 Nitro 内部的 Vue 服务端渲染端点**。整节就这一句，默认值没有声明。

## `externalVue`：默认开，但已经写了"不久会被移除"

它做的事：构建时把 `vue`、`@vue/*` 和 `vue-router` 外置。默认开，可以关。

要紧的是那条警告，原文只有一句：**这个特性很可能在不久的将来被移除。**这一页上很多开关都挂着"可能被移除或修改"的总说明，但**像这样单独对某个开关写"即将移除"的只有一处**。

## `viteEnvironmentApi`：一改就动插件注册方式

作用：启用 Vite 6 的新 Environment API，用于改善构建配置与插件架构。`future.compatibilityVersion` 设成 `5` 时默认启用，也可以显式开启来做测试。

文档给的收益是三点：开发与生产构建之间更好的一致性、对"按环境区分的配置"更细的控制、性能改善。

那条重要提示是这一节的关键：**启用这个特性会改变 Vite 插件被注册与配置的方式**，并指向 Nuxt 5 升级指南里的 Vite Environment API 迁移章节。也就是说它是会影响你现有 `addVitePlugin` 代码的行为变更，而不是纯内部优化。

## `enforceModuleCompatibility`：让不兼容的模块直接失败

原文那句问话：当某个 Nuxt 模块不兼容时，Nuxt 是否应该**抛错并加载失败**。默认关闭。

它与 `useLogger` 那条"setup 超过 1 秒 Nuxt 会警告"属于同一类"把隐患暴露出来"的开关——**关掉时不兼容的模块照常跑**，开了以后在启动阶段就把问题拦下。文档没有列出"不兼容"的判定来源（版本约束写在模块 `meta.compatibility`，但这一节没有把它和这个开关连起来说明）。

## `typescriptPlugin`：它其实是在装一个外部模块

作用：用 `@dxup/nuxt` 模块启用增强的 TypeScript 开发体验，原文称它提供更好的 TypeScript 集成与开发工具。默认关闭。

两个前提写在重要提示里，缺一个就不工作：**`typescript` 要作为依赖安装**、**VS Code 要配置成使用工作区的 TypeScript 版本**（指向 VS Code 文档）。进一步阅读指向 `github.com/KazariEX/dxup`——**这个模块不是 Nuxt 仓库里的东西**，开这个开关等于把一个第三方开发工具引入构建链。

## `decorators`：注意它支持的是哪一版装饰器

作用：在整个 Nuxt/Nitro 应用里启用装饰器语法。三条机制说明：

- 用 Vite 构建器（默认）时，装饰器通过 Babel 的 `@babel/plugin-proposal-decorators` 降级；用 webpack 或 rspack 时通过 esbuild 降级。
- TypeScript 长期通过 `compilerOptions.experimentalDecorators` 支持装饰器，**那套实现早于 TC39 标准化流程**；现在装饰器是 Stage 3 提案，TS 5.0+ 无需特殊配置就支持。
- **最关键那句用粗体写着：启用 `experimental.decorators` 支持的是 TC39 提案，不是 TypeScript 旧的 `experimentalDecorators` 实现。**

配套警告：**在它最终进入 JS 标准之前仍可能有改动**。

用 Vite 构建器或 Nitro 服务端构建时，你需要**额外把两个 Babel 包装成 devDependency**：`@babel/plugin-proposal-decorators` 与 `@babel/plugin-syntax-jsx`（npm/pnpm/yarn 三条命令都给了，**没有 bun 与 deno 那两条**）。文档补了一条 tip：如果这两个包不存在，**Nuxt 会提示你自动安装**。

官方示例是最小形状：

```ts [app/app.vue]
function something (_method: () => unknown) {
  return () => 'decorated'
}

class SomeClass {
  @something
  public someMethod () {
    return 'initial'
  }
}

const value = new SomeClass().someMethod()
// this will return 'decorated'
```

## 十个开关的默认值

| 开关 | 默认 | 备注 |
| --- | --- | --- |
| `watcher` | `chokidar-granular` | `compatibilityVersion: 5` 时默认变 `'builder'` |
| `buildCache` | 关 | 九个文件绕过它、Nitro 永远重建、最多 10 份 tarball |
| `checkOutdatedBuildInterval` | 未声明数值 | `appManifest: false` 时禁用；`false` 也关 |
| `appManifest` | 开 | 客户端遵守路由规则的机制 |
| `noVueServer` | 未声明 | 关掉 Nitro 里的 Vue 服务端渲染端点 |
| `externalVue` | 开 | **单独被标注"很可能不久被移除"** |
| `viteEnvironmentApi` | `compatibilityVersion: 5` 时开 | 改变插件注册方式，须看迁移章节 |
| `enforceModuleCompatibility` | 关 | 开了会让不兼容模块直接加载失败 |
| `typescriptPlugin` | 关 | 依赖 `typescript` 与 VS Code 工作区版本设置 |
| `decorators` | 未声明 | 支持 TC39 而非 `experimentalDecorators`；要装两个 Babel 包 |

下一篇把剩下的开关一次列完：`asyncContext`、`restoreState` 之外那十一条里没进前面几组的（`defaults`、`cookieStore`、`sharedPrerenderData`、`clientNodeCompat`、`early404`、`entryImportMap`、`browserDevtoolsTiming`、`debugModuleMutation` 等），以及它们各自属于哪一类。

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/experimental-features
