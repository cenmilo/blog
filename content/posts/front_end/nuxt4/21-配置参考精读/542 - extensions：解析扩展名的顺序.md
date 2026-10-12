---
url: /posts/05d96cb5a1c6cc9dbd87a29beb56afeb/
title: extensions：解析扩展名的顺序
date: 2027-04-10T11:00:00+08:00
updated: 2027-04-10T11:00:00+08:00
author: cmdragon

summary:
  摘要：顶层 `extensions` 只有一句定义——**"The extensions that should be resolved by the Nuxt resolver."**（Nuxt 解析器要解析的扩展名），类型 `array`，默认六项：`.js`、`.jsx`、`.mjs`、`.ts`、`.tsx`、`.vue`。这份清单被三个地方直接引用：页面文件"可以是 Nuxt 支持的任何有效扩展名"、组件目录里"默认情况下凡扩展名在 `extensions` 键里的文件都被当成组件"、以及 Kit 的路径解析函数（`resolvePath` 会遵守 Nuxt 的 alias 与 extensions 选项）。要注意它和 `vite.resolve.extensions` **不是同一份清单**：后者默认是 `.mjs`、`.js`、`.ts`、`.jsx`、`.tsx`、`.json`、`.vue` 七项，多了 `.json` 且顺序不同。文档没有讨论同名多扩展并存时按什么顺序选中。

categories:
  - 前端开发

tags:
  - nuxt4
  - 解析
  - extensions
  - 组件
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

目录里同时放了 `Button.vue` 和 `Button.tsx`，最后被注册成组件的是哪一个？

这一节只有两行，答不了这个问题。但它定义的那份清单，决定了三处行为。

## 这一节的全部

`## extensions`：

> The extensions that should be resolved by the Nuxt resolver.

- **类型**：`array`
- **默认**：

```json
[
  ".js",
  ".jsx",
  ".mjs",
  ".ts",
  ".tsx",
  ".vue"
]
```

六项，**每项都带点号**，顺序就是文档给出的顺序。这一节到此结束——**没有示例，没有说明顺序有什么用**。

## 谁在引用这份清单

它的引用点在别处，一共三处，都是行为定义级别的。

**页面文件。** `directory-structure/app/pages` 第一句：**"Pages are Vue components and can have any valid extension that Nuxt supports (by default `.vue`, `.js`, `.jsx`, `.mjs`, `.ts` or `.tsx`)."** 那句里的"有效扩展名"链接回的正是 `api/nuxt-config#extensions`。

也就是说：**页面后缀的合法集合由这个键定义**。同一组扩展名在两页出现，顺序写法不同（这页是 `.js` 开头，pages 页把 `.vue` 列在首位），但成员一致——六项。

**组件目录。** `directory-structure/app/components` 里写得更直接：**"By default, any file with an extension specified in the extensions key of `nuxt.config.ts` is treated as a component."**，紧接着给收窄的办法：

```ts
export default defineNuxtConfig({
  components: [
    {
      path: '~/components',
      extensions: ['.vue'], // 只把 .vue 当组件
    },
  ],
})
```

同页还有一句相关的：**"If `pattern` is specified, the `extensions` option has no effect, so make sure your pattern matches the file extensions you want to scan."**——**一旦给目录声明写了 `pattern`，`extensions` 就被忽略**，扫描范围完全由那个 glob 决定。这是本节最容易踩的联动。

**Kit 的路径解析。** `api/kit/resolving` 那页把这份清单当成解析器的默认配置：`resolvePath` 的描述是 **"A function that resolves a path relative to the base path and respects Nuxt alias and extensions options."**，其参数表里 `extensions` 一项写的是 `string[]`、可选（`false` 表示未传），说明列是 **"The file extensions to try. Default is Nuxt configured extensions."**

这句"Default is Nuxt configured extensions"是两处清单之间关系的唯一官方表述：**模块代码里做路径解析时，默认用的就是这个顶层键的值**；要临时改，就通过参数传，而**参数传的是"要试哪些扩展名"的完整列表**（表里没写它是追加还是替换）。

## 别和 vite.resolve.extensions 混

配置参考里还有另一个同名条目，在 `vite` 节下：`vite.resolve.extensions`，默认七项：

```json
[
  ".mjs",
  ".js",
  ".ts",
  ".jsx",
  ".tsx",
  ".json",
  ".vue"
]
```

两份的差异是实打实的：

| 清单 | 成员数 | 有 `.json` 吗 | 首项 |
| --- | --- | --- | --- |
| 顶层 `extensions` | 6 | **没有** | `.js` |
| `vite.resolve.extensions` | 7 | **有** | `.mjs` |

**顶层那份不含 `.json`。** 这个差别对得上前面那三处引用：页面、组件、Kit 解析用的都是顶层那份；`.json` 文件能被 import 是 Vite 侧解析链的事，不是"扩展名在顶层清单里"的结果。

反过来也成立：**只改顶层 `extensions` 不会自动改 Vite 侧那份**，两处在文档里是两个独立条目，各自给默认值，官方没有任何一句说明它们会同步。要两边一致，就得写两次——这一条属于"文档未说明、需自行验证"的范围。

## 顺序问题：文档没答

回到开头那个例子：同名不同扩展的两个文件都在目录里，注册的是哪一个。

**文档没有回答。** 这一节只说这些扩展名"should be resolved by the Nuxt resolver"、给了数组顺序，但**既没有说数组顺序就是尝试顺序，也没有说同名冲突时怎么选**。pages 那节用 "any valid extension" 描述可能性，components 那节用 "any file with an extension specified in" 描述判定条件——两处都不涉及优先级。

所以别把默认数组的顺序当成规则写进规范。想确认自己的项目里会发生什么，最短的做法是在同一个目录下放同名两份文件（例如 `Probe.vue` 与 `Probe.tsx`），构建后看注册出来的是哪一个，再据此决定要不要用 `extensions: ['.vue']` 收窄。

## 自查

- 顶层 `extensions` 是 `array`，默认六项 `.js`/`.jsx`/`.mjs`/`.ts`/`.tsx`/`.vue`，本节无示例、也没说明顺序的含义。
- 三处在引用它：页面合法后缀、组件目录的默认判定（"any file with an extension specified in the extensions key"）、Kit 的 `resolvePath`（默认用 Nuxt 配置的扩展名）。
- **组件目录声明里一旦写了 `pattern`，`extensions` 就失效**；要收窄组件后缀用扩展形式的 `extensions: ['.vue']`。
- `vite.resolve.extensions` 是另一份清单（七项、含 `.json`、首项 `.mjs`），与顶层那份不同步，官方没说会联动。
- 同名多扩展的优先级文档未给；要定规则就实测一次再写进团队约定。

## 下一篇

下一篇 543 读 `ignore`：那份默认数组列的是 Nuxt 不扫描的路径，官方对它的定位是"扩展或覆盖"，并直接把你指向 chokidar 的文档。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## extensions` 整节：定义句、Type `array` 与六项默认 JSON；`### vite.resolve` 下 `#### extensions` 的 Type 与七项默认 JSON（含 `.json`）；`### vite.root` 的默认 `"/<rootDir>"`）
- https://nuxt.com/docs/4.x/directory-structure/app/pages（"Pages are Vue components and can have any valid extension that Nuxt supports"与其列举）
- https://nuxt.com/docs/4.x/directory-structure/app/components（"By default, any file with an extension specified in the extensions key of nuxt.config.ts is treated as a component"、扩展形式里 `extensions: ['.vue']` 的示例与用途句、"If `pattern` is specified, the `extensions` option has no effect"）
- https://nuxt.com/docs/4.x/api/kit/resolving（`resolvePath` 遵守 alias 与 extensions 的描述、参数表中 `extensions` 的类型/可选性/"Default is Nuxt configured extensions"）
