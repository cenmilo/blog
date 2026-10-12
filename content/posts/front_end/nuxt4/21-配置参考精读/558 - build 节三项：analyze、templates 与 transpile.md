---
url: /posts/56de57f2fca447983ea51a96478b3aef/
title: build 节三项：analyze、templates 与 transpile
date: 2027-04-16T09:00:00+08:00
updated: 2027-04-16T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`build` 这一节的定性只有两个词：**"Shared build configuration."**——没说"shared"是在谁之间共享。下面三项各有各的问题：`analyze` 的默认值是一份三项 JSON（`template: "treemap"`、`projectRoot`、`filename: "/<rootDir>/.nuxt/analyze/{name}.html"`），**但官方示例写的是顶层 `analyze:` 而不是 `build.analyze:`**；`templates` 的说明是**建议你别用它**——"It is recommended to use `addTemplate` from `@nuxt/kit` instead of this option."；`transpile` 给了四种元素形态（包名、函数、字符串、匹配文件名的正则对象），**可那句"函数会收到一个对象"后面的括号在文档里是空的**，唯一出现的参数名 `isLegacy` 只在示例里。

categories:
  - 前端开发

tags:
  - nuxt4
  - build
  - transpile
  - 打包分析
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

依赖里的新语法把构建搞挂了，你搜到的第一条建议是"把它加进 `build.transpile`"。

问题是：那个函数参数到底收什么？这一节自己也答不上来。

## 一、build 节的定性只有两个词

`## build` 下面唯一的一句说明是 **"Shared build configuration."**，然后直接进子项。

**"shared" 指什么、和谁共享，文档没有任何解释。** 同层还有个 `## builder`（客户端构建器选择）、`### server.builder`（服务端构建器），三者名字相近但分工不同：这一节是**跨构建器的构建期能力**（分析、模板生成、转译），从三项的内容看确实都是"两边都用得上"的东西——但这个归纳是本系列的读法，**不是文档原话**。

## 二、analyze：默认值在 build 下，示例却写在顶层

`### analyze` 的说明："Nuxt allows visualizing your bundles and how to optimize them."，然后是那句与 webpack 那节逐字相同的选项说明——**设 `true` 开启，或传对象**（webpack 侧给 webpack-bundle-analyzer 的选项、vite 侧给 rollup-plugin-visualizer 的选项）。

- **类型**：`object`
- **默认**：

```json
{
  "template": "treemap",
  "projectRoot": "/<rootDir>",
  "filename": "/<rootDir>/.nuxt/analyze/{name}.html"
}
```

三行各自有用：**`template` 默认 `treemap`**（看体积图的那块图）、**`projectRoot` 是项目根**、**`filename` 决定报告写到 `.nuxt/analyze/{name}.html`**。最后这个 `{name}` 对应命令侧 `nuxt analyze --name` 的值（默认 `default`），所以不传参数时文件名是 `default.html`。

问题在示例的位置。官方给的是：

```ts
export default defineNuxtConfig({
  analyze: {
    analyzerMode: 'static',
  },
})
```

**这一行写在顶层，不是 `build.analyze`。** 而同一份默认值与同样的示例，在 `webpack.analyze` 那节又是写在 `webpack` 下面、并明确说"If you are using webpack..."。

结论只能是：**同一个分析功能在文档里出现了三个位置（顶层 `analyze`、`build.analyze`、`webpack.analyze`），官方没有解释三者关系。** 要确认你的写法生效，最省事的判断是**跑一次构建看有没有生成报告文件**：`npx nuxt analyze` 或直接 `npx nuxt build` 之后检查 `.nuxt/analyze/` 目录里有没有 html。带 `--name` 时文件名会变，这也是校验位置对不对的办法。

另外记一条命令侧事实：**`nuxt analyze` 会构建并分析生产包，构建完默认起一个本地服务把结果展示出来——除非你传 `--no-serve` 或环境里设了 `CI`**；而预渲染在分析时不会跑（要就加 `--prerender`）。

## 三、templates：官方直接建议你用 Kit

`### templates` 的说明句是一句劝退：

> It is recommended to use `addTemplate` from `@nuxt/kit` instead of this option.

- **类型**：`array`

示例仍给了，两个字段各带一行注释：

```ts
export default defineNuxtConfig({
  build: {
    templates: [
      {
        src: '~~/modules/support/plugin.js', // `src` can be absolute or relative
        dst: 'support.js', // `dst` is relative to project `.nuxt` dir
      },
    ],
  },
})
```

两行注释就是全部规则：**`src` 可绝对可相对，`dst` 相对 `.nuxt` 目录**。

这一项的性质与 `analyze` 不同——**它不是"你少用"，而是"这个能力有更合适的入口"**：模块作者要用 Kit 的 `addTemplate`。也就是说 `build.templates` 更像是留给配置侧的一条旧通道，**而为什么还留着它，本节没说**。

## 四、transpile：四种元素，和一处空白

`### transpile` 的说明两句：

> If you want to transpile specific dependencies with Babel, you can add them here. Each item in transpile can be a package name, a function, a string or regex object matching the dependency's file name.
>
> You can also use a function to conditionally transpile. The function will receive an object ().

- **类型**：`array`

第一句给出四种元素形态：

| 形态 | 官方措辞 | 判定对象 |
| --- | --- | --- |
| 包名 | a package name | 依赖包 |
| 函数 | a function | （见下） |
| 字符串 | a string ... matching the dependency's **file name** | 文件名 |
| 正则对象 | regex ... matching the dependency's **file name** | 文件名 |

**第二句的括号里是空的**："The function will receive an object ()。" ——**文档自己没写这个对象有什么字段**。这一点必须按"文档未记载"处理，别按别的教程的记忆补字段名。

能确证的只有示例里出现过的那一个：

```ts
export default defineNuxtConfig({
  build: {
    transpile: [({ isLegacy }) => isLegacy && 'ky'],
  },
})
```

**`isLegacy` 是全套 4.x 文档里唯一出现过的参数名**（在 Kit 的示例里也见过按 `builder === '@nuxt/webpack-builder'` 条件 push 的写法，那是模块侧用法）。要确认自己的构建器实际传了什么，做法是在配置里打一行日志：

```ts
transpile: [
  (options) => {
    console.log('[probe] transpile fn receives', Object.keys(options))
    return false
  },
],
```

**这段是本系列给的取证写法，官方没有。** 返回值的形状同样只有一个证据：示例里返回的是 `'ky'` 这个包名或 `false`，**"能返回哪些值"本节没写**。

还有一条与 `transpile` 直接相关的错误码：`NUXT_E1006` 说 `onPrehydrate()` 若来自未被转译的依赖就会报，**Resolution 是把那个库加进 `build.transpile`**——这是文档里唯一一处"必须动这个键才能解决"的场景。

## 自查

- `## build` 的定性只有 "Shared build configuration." 一句，"shared" 指什么文档未解释；下面是 `analyze`、`templates`、`transpile` 三项。
- `analyze` 的默认三项 JSON 说明"报告写到 `.nuxt/analyze/{name}.html`、默认 treemap 模板"；但**同一功能在文档里出现于顶层 `analyze`、`build.analyze`、`webpack.analyze` 三处，关系未说明**，以能否生成报告文件为准。
- `templates` 官方直接建议改用 Kit 的 `addTemplate`；留下的规则只有"`src` 可绝对可相对、`dst` 相对 `.nuxt`"。
- `transpile` 支持包名／函数／字符串／正则四种元素，**后两种匹配的是依赖的文件名**；那句"函数收到一个对象"的括号是空的，`isLegacy` 只是示例里出现过的字段。
- 唯一"必须动 `build.transpile`"的官方场景是 E1006（`onPrehydrate()` 来自未转译依赖）。

## 下一篇

下一篇 559 读 `sourcemap`：它接受一个对象（`server`／`client`），也可用单个布尔同时控制两端，三种取值 `true`／`false`／`'hidden'` 各自的差别官方逐条写了——默认是服务端 `true`、客户端 `false`。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## build` 的定性句与三个 `###` 子项全文：`analyze` 的两句说明、`object` 类型与三项默认 JSON、顶层 `analyze` 示例；`templates` 的 addTemplate 建议句、`array` 类型与 `src`/`dst` 注释；`transpile` 的两句说明（含空括号那句）、`array` 类型与 `isLegacy` 示例）
- https://nuxt.com/docs/4.x/api/nuxt-config（`### webpack.analyze` 的同一份默认 JSON 与 `webpack.analyze` 示例，用于三处位置对照）
- https://nuxt.com/docs/4.x/api/commands/analyze 与 https://nuxt.com/docs/4.x/api/commands/build（`--name` 默认 `default` 与报告文件名、`--no-serve`／设了 `CI` 时不起服务、分析时不预渲染）
- https://nuxt.com/docs/4.x/errors/e1006 与 https://nuxt.com/docs/4.x/api/kit/templates、https://nuxt.com/docs/4.x/api/kit/plugins（把库加进 `build.transpile` 这条 Resolution；`addTemplate` 作为 `build.templates` 的官方替代入口；Kit 示例里按 builder 条件 push 的写法）
