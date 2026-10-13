---
url: /posts/d50fd6e2ac71f9eebaac9a27de2ac9a3/
title: 675 - import.meta 在 Nuxt 里的八个运行时字段与两个构建期字段
date: 2027-05-25T09:00:00+08:00
updated: 2027-05-25T09:00:00+08:00
author: cmdragon

summary:
  摘要：这一节把 `import.meta` 上 Nuxt 注入的字段分成两组，**两组的性质完全不同**。运行期（App）那组有**八个**：`client`、`browser`、`server`、`nitro`、`dev`、`envName`、`test`、`prerender`，页面给的关键句是**"这些值是静态注入的，可以用来对运行时代码做 tree-shaking"**；构建期那组只有**两个**：`import.meta.env`（**等于 `process.env`**）与 `import.meta.url`（**当前文件的可解析路径**），并且明确写"**这两项在模块里和 `nuxt.config` 里都可用**"。读表时注意两对同义描述：**`client` 与 `browser` 的描述逐字相同、`server` 与 `nitro` 也逐字相同**；`envName` 带 v4.5 标记且包含通过 `--envName` 传入的自定义值。

categories:
  - 前端开发

tags:
  - nuxt4
  - import.meta
  - 条件编译
  - tree-shaking
  - 环境变量
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`if (import.meta.server)` 这行你在组件里见过很多次。这一节讲它的来历：**哪些字段是构建时替换掉的、哪些只在配置阶段存在**，这决定了同一段代码在打包后是什么形状。

## 这一节先给的定位

**用 ES 模块时，你可以从"导入或编译你的 ES 模块的那段代码"那里拿到一些元信息，这件事通过 `import.meta` 完成——它是一个为你的代码提供这些信息的对象**。

然后一句解释了为什么文档到处在用它：**整份 Nuxt 文档里你会看到用它的片段，用来判断代码此刻是跑在客户端还是服务端**。

这一节把字段切成两组，**分组依据不是功能而是"谁能拿到"**。

## 运行期八个：静态注入

这一组的定性句最关键：

> These values are statically injected and can be used for tree-shaking your runtime code.

**静态注入**意味着这些布尔值在编译时就被替换成字面量，于是 `if (import.meta.server) { ... }` 里那一支在客户端构建中是**死代码，会被摇掉**。这是"用哪个写法判断跑在哪侧"的正式答案。

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `import.meta.client` | boolean | 在客户端求值为 true |
| `import.meta.browser` | boolean | 在客户端求值为 true |
| `import.meta.server` | boolean | 在服务端求值为 true |
| `import.meta.nitro` | boolean | 在服务端求值为 true |
| `import.meta.dev` | boolean | 运行在 Nuxt 开发服务器上时为 true |
| `import.meta.envName` | string | 当前 Nuxt 环境名，**包含通过 `--envName` 传入的自定义值**（带 v4.5 标记） |
| `import.meta.test` | boolean | 在测试上下文里运行时为 true |
| `import.meta.prerender` | boolean | **在你构建的预渲染阶段、于服务端渲染 HTML 时**为 true |

两处要留意：

**两对描述逐字相同**：`client` 与 `browser` 都是"在客户端求值为 true"，`server` 与 `nitro` 都是"在服务端求值为 true"。**页面没有说这四者何时该选哪一个**，也没说别名存在的理由——只能当作同义写法。

**`prerender` 那句限定很有用**：它是"预渲染阶段在服务端渲染 HTML"时为真，**与 `server` 的差别在于它只覆盖预渲染那一段**，普通 SSR 请求期间不成立。**配合 `server` 才能区分"预渲染时"与"每次请求时"**，这正是决定要不要做一次性输出（比如生成路由清单）的判据。

## 构建期两个：模块与配置里可用

这一组的定性是：**这两项在模块里和在你的 `nuxt.config` 里都可用**。

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `import.meta.env` | object | **等于 `process.env`** |
| `import.meta.url` | string | **当前文件的可解析路径** |

两条各有实际含义：

**`import.meta.env` 是 `process.env` 的对象**——注意它被归在"构建期"这一组，**页面没有说它在运行时代码里是否同样可用**，所以**要在浏览器里读环境值，正规路径是 runtime config 而不是它**。

**`import.meta.url` 是全页唯一一个给了完整示例的字段**，示例正是模块里那句标准写法：

```ts [modules/my-module/index.ts]
import { createResolver } from 'nuxt/kit'

// Resolve relative from the current file
const resolver = createResolver(import.meta.url)

export default defineNuxtModule({
  meta: { name: 'myModule' },
  setup () {
    addComponent({
      name: 'MyModuleComponent',
      // Resolves to '/modules/my-module/components/MyModuleComponent.vue'
      filePath: resolver.resolve('./components/MyModuleComponent.vue'),
    })
  },
})
```

这段里有三处与前面章节直接对上：

**第一，导入面写的是 `nuxt/kit`**（本地模块可以直接用这个助手子路径导入），**这意味着"不必把 `@nuxt/kit` 加进项目依赖"**——这句解释就在 `modules/` 目录那一节的示例注释里。

**第二，`createResolver(import.meta.url)` 的作用被注释点明**："相对当前文件解析"。**结果路径也写死了**：`./components/MyModuleComponent.vue` → `/modules/my-module/components/MyModuleComponent.vue`。**这就是"本地模块里的相对路径以自己的文件为基准"的具体证据**，与图层那一节"层里的相对路径按用户项目解析"正好相反。

**第三，`meta: { name: 'myModule' }`**——最小模块定义只需要 `meta` 加 `setup`。

## 这些字段与别的判断方式的关系

这一节**只讲 `import.meta`，没有比较其他写法**。能确定关联的有三处：

| 判断需求 | 这一节给的 | 页面之外的对应事实 |
| --- | --- | --- |
| 哪一侧 | `client`／`browser`／`server`／`nitro` | 服务端运行时另有钩子表（Nitro 那九行），两者不是一个层面的东西 |
| 是否开发期 | `import.meta.dev` | 配置项 `dev` 与 CLI 的 `--dev`；`builder:watch` 也只在开发期触发 |
| 是否测试 | `import.meta.test` | 配置项 `test` 在配置参考里只有一句"`boolean`/默认 `false`"，**`nuxt.options.test` 全库没有读取处**——所以 `import.meta.test` 是这一族里少数被写成公开入口的那个，**但页面没有把它与 `test` 配置项对应起来** |
| 哪个环境 | `import.meta.envName` | 编程式加载配置时也有 `envName` 选项，且**外部传入优先于 `nuxt.config` 里写的**——与这里"包含 `--envName` 传入的自定义值"方向一致 |

## 用在哪、注意什么

**要 tree-shaking，就用这一族的布尔值做分支**：**静态注入是它的全部优势**，写成 `import.meta.server` 而不是运行时才计算的条件。

**判断"只预渲染时做一件事"，用 `prerender`**，它是八个里唯一带阶段限定的。

**别把 `import.meta.env` 当运行时的环境读取通道**：这一节把它归在构建期组，且它的定义就是 `process.env`——**服务端进程的环境变量**。要给浏览器用，走 runtime config 的 `public` 一侧（并且**不要把私有 API key 放进去，它会进公开 bundle**）。

这一节未记载的还有：这些字段在 Nitro 服务端代码里是否同样注入（**运行期那组的措辞是 "Runtime (App)"，指向应用侧**）、`envName` 的确切取值集合与默认值、以及 `nitro` 与 `server` 是否存在任何时点上的差别（描述相同但名字不同）。

下一篇进入图层：先看它的定义与用途清单。

## 下一篇

下一篇 676：图层概念：七种用途、两个版本下限与那两处清单差别

## 本文依据

- https://nuxt.com/docs/4.x/api/advanced/import-meta
- https://nuxt.com/docs/4.x/directory-structure/modules
