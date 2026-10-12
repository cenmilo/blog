---
url: /posts/32d518cd5e364282ff00a1b525c59e85/
title: esbuild 那一层 options 与四项共享默认值
date: 2027-04-28T11:00:00+08:00
updated: 2027-04-28T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## esbuild` 标题下**没有定性句**，直接进 `### options`，那句定性是写给 `options` 这一层的：**"Configure shared esbuild options used within Nuxt and passed to other builders, such as Vite or webpack."**——多的一层 `options` 是这一节的形状特征。四个子项的默认值是 `jsxFactory: "h"`、`jsxFragment: "Fragment"`、`target: "esnext"`，而第四个 **`tsconfigRaw` 只有 `**Type**: object`，连默认值都没有**。同一份默认值在别处以**扁平对象**形式又出现两次：`vite.esbuild`（**无说明句**）与 `webpack.loaders.esbuild`（多一条指向 esbuild-loader 的 See），三份的 `tsconfigRaw` 在那里都是 `{}`。**这三处会不会互相同步、改顶层那份要不要同时改另外两份，文档没有写。**

categories:
  - 前端开发

tags:
  - nuxt4
  - esbuild
  - JSX
  - 构建器共享项
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

配置自动导入的 JSX 工厂函数，或者把产物目标降到 `es2020`，你搜 `esbuild` 会找到三处：顶层一个、`vite` 里一个、`webpack.loaders` 里一个。三份长得像，形状却不一样。

## 顶层这一节的形状

`## esbuild` 下面**没有任何定性句**，第一个子项就是 `### options`，而定性句写在它那里：

> Configure shared esbuild options used within Nuxt and passed to other builders, such as Vite or webpack.

所以顶层的形状是 `esbuild.options.<键>`，**比另外两处多一层 `options`**。四个 `####` 子项逐条如下：

| 子项 | 说明句 | Type | Default |
| --- | --- | --- | --- |
| `jsxFactory` | **无** | `string` | `"h"` |
| `jsxFragment` | **无** | `string` | `"Fragment"` |
| `target` | **无** | `string` | `"esnext"` |
| `tsconfigRaw` | **无** | `object` | **无默认值行** |

四个子项**都没有说明句**，这一节的正文一共只有 `options` 那一句。`## esbuild` 也**没有 See**——esbuild 自己的选项语义（`target` 能填哪些档位、`tsconfigRaw` 接受什么结构）都在 esbuild 文档里，本节一概没带链接。

`tsconfigRaw` 是唯一连默认值都不给的子项：类型写 `object`，其余无。而在下面两处构建器版本里，它的默认值都是 `{}`——**同一份配置在三处的记载完整度不同**。

## 另外两份的形状

`vite.esbuild` 这一条**标题下直接是类型与默认值，一句说明都没有**：

```json
{
  "target": "esnext",
  "jsxFactory": "h",
  "jsxFragment": "Fragment",
  "tsconfigRaw": {}
}
```

`webpack.loaders.esbuild` 给的是**逐字相同的默认值**，只多了一条 See：

```json
{
  "target": "esnext",
  "jsxFactory": "h",
  "jsxFragment": "Fragment",
  "tsconfigRaw": {}
}
```

那条 See 指向 `esbuild-loader`（webpack 侧的实现），而 `vite` 那一份没有给外链——两处对同一个对象的说明待遇不同。

三份放一起，能整理出的事实是这些：

| | 顶层 `esbuild` | `vite.esbuild` | `webpack.loaders.esbuild` |
| --- | --- | --- | --- |
| 形状 | `esbuild.options.*`（多一层） | 直接对象 | 直接对象 |
| 定性句 | 有（写在 `options` 上） | **无** | **无** |
| 默认值 | 四项拆开写，`tsconfigRaw` 无默认 | 一份 JSON，含 `tsconfigRaw: {}` | 与左侧逐字相同 |
| See | 无 | 无 | esbuild-loader |

## 共享那句到底承诺了什么

顶层那句里的两个词是这一节的全部机制说明：**shared**（Nuxt 内部共用）与 **passed to other builders**（会传给别的构建器，例如 Vite 或 webpack）。

按字面读，顶层这份是"源头"、两处构建器版本是"被传过去的值"。但文档**没有写这三者之间的同步规则**，所以两个实际操作问题都没有依据：

1. **在 `vite.esbuild` 里改 `target`，会不会覆盖顶层那份？** 没有记载。
2. **顶层改了，两处构建器需不需要跟着改？** 没有记载——`builder` 那一节讲的是选哪个构建器，`## esbuild` 这节只说了"会传过去"。

要落地只能实测：改一处、跑构建、看产物的目标语法或 JSX 工厂调用有没有变。**不要假定三份会自动一致**，因为文档既没说它们同步、也没说它们互不干涉。

另外一句限制值得记住：`target` 默认 `"esnext"` 意味着**默认不做语法降级**；要给旧浏览器降级就得显式改这一项。这是从默认值本身读出来的结论，本节没有给可选档位清单（哪些字符串合法属 esbuild 侧定义）。

## 自查

- `## esbuild` 自身无定性句；定性写在 `### options` 上，形状是**多一层 `options`**。
- 四个子项**全无说明句**：`jsxFactory "h"`、`jsxFragment "Fragment"`、`target "esnext"`、`tsconfigRaw`（`object`，**无默认值**）。
- 顶层无 See；`vite.esbuild` 无说明句；`webpack.loaders.esbuild` 默认值相同但多一条 esbuild-loader 的 See。
- 两份构建器默认里 `tsconfigRaw` 是 `{}`，与顶层"无默认值行"不一致——**记载完整度差异，不是值不同**。
- 三处之间的同步／覆盖关系**文档未写**；`target` 默认 `esnext` 即默认不降级，合法档位不在本节。

## 下一篇

下一篇 597 把这一分区散见的因果句汇成一张清单：哪些配置项被文档明确连在一起（`app.buildAssetsDir` 相对 `baseURL` 或 `cdnURL`、`typesDir` 默认跟随 `buildDir`、`typescript.builder` 由 `builder` 推断、`features.inlineStyles` 只在用 Vite 时可用、`ssr: false` 是 `spaLoadingTemplate` 与 `spaLoadingTemplateLocation` 的生效前提、`experimental.inlineRouteRules` 决定 `defineRouteRules` 能不能用），以及**哪些看起来有关系、文档却始终没连起来的**。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## esbuild` 无定性句的事实、`### options` 那句 shared/passed 原文、四个 `####` 子项的类型与默认值、`#### tsconfigRaw` 无默认值行；`### vite.esbuild` 的 `object` 与四项默认 JSON 及无说明句；`#### webpack.loaders.esbuild` 的相同 JSON 与 esbuild-loader 的 See；`### server.builder` 用于对照"构建器选择"）
