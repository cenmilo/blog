---
url: /posts/adcb2c450adf4924171f0d880e4c2ab7/
title: plugins 与 modules：两种数组的形状、顺序与去重
date: 2027-04-24T10:00:00+08:00
updated: 2027-04-24T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：这两节都是数组，但规则不一样。`## plugins` 有 `**Type**: array`、**没有默认值行**，条目可以是字符串或 `src`＋`mode` 对象，字符串结尾是 `.client` 或 `.server` 时**只在对应上下文加载**；那条 `::callout` 给出两个常被踩的规则——`~/plugins` 目录本来就自动注册、**只有要定制顺序时才需要写进配置**，以及**所有插件按 src 路径去重**。`## modules` 的条目有三种形状（包名或路径字符串、`[模块, 选项对象]` 元组、内联函数），解析顺序是先按 node require 路径找、用到 `~~` 别名时再从 `rootDir` 解析，而它的 callout 讲的是**顺序**：先执行配置里定义的模块，再执行 `modules/` 目录里的、按字母序。

categories:
  - 前端开发

tags:
  - nuxt4
  - 插件注册
  - 模块注册
  - 执行顺序
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

同一个文件在配置里写了两遍，结果只加载一次；另一个数组里换了顺序，两个模块的行为就变了——`plugins` 与 `modules` 都是数组，但它们的规则不是一套。

## plugins：三种条目形状与一条去重规则

`## plugins` 的定性句是 "An array of nuxt app plugins."，紧接着是形状说明：

> Each plugin can be a string (which can be an absolute or relative path to a file). If it ends with `.client` or `.server` then it will be automatically loaded only in the appropriate context.
> It can also be an object with `src` and `mode` keys.

- **Type**: `array`
- **没有 Default 行。**

那条 `::callout` 是这一节信息量最大的地方：

> **Note**: Plugins are also auto-registered from the `~/plugins` directory and these plugins do not need to be listed in `nuxt.config` unless you need to customize their order. All plugins are deduplicated by their src path.

三件事在里面：目录里的插件**默认自动注册**；把它们再写进配置的唯一理由是**要定制顺序**；以及**按 src 路径去重**——这就是"同一文件写两遍只加载一次"的依据。See 指向 `app/plugins` 目录那一页。

示例给了六条，覆盖两种形状与三种上下文：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  plugins: [
    '~/custom-plugins/foo.client.js', // only in client side
    '~/custom-plugins/bar.server.js', // only in server side
    '~/custom-plugins/baz.js', // both client & server
    { src: '~/custom-plugins/both-sides.js' },
    { src: '~/custom-plugins/client-only.js', mode: 'client' }, // only on client side
    { src: '~/custom-plugins/server-only.js', mode: 'server' }, // only on server side
  ],
})
```

两种指定上下文的方式**在同一个示例里并存**：文件名后缀 `.client.js`，或者对象里写 `mode: 'client'`。文档没有说这两者是否完全等价、也没有说同时出现时以谁为准；示例只把后缀形式用在字符串条目上、把 `mode` 用在对象条目上。另外，**没有后缀的字符串条目被注释成 "both client & server"**，这就是"默认两端都跑"的依据。

`mode` 的合法取值，本节能确证的只有示例里出现的 `client` 与 `server`（`all` 这个写法在这一节没有出现过）。

## modules：三种条目形状与一条顺序规则

`## modules` 的定性句把模块定位成扩展点："Modules are Nuxt extensions which can extend its core functionality and add endless integrations."

条目形状有三种，写在同一句里：

> Each module is either a string (which can refer to a package, or be a path to a file), a tuple with the module as first string and the options as a second object, or an inline module function.

解析规则紧随其后：

> Nuxt tries to resolve each item in the modules array using node require path (in `node_modules`) and then will be resolved from project `rootDir` if `~~` alias is used.

- **Type**: `array`
- **没有 Default 行。**

示例四条，与上面三种形状一对应（第四条是内联函数）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  modules: [
    // Using package name
    '@nuxt/scripts',
    // Relative to your project rootDir
    '~~/custom-modules/awesome.js',
    // Providing options
    ['@nuxtjs/google-analytics', { ua: 'X1234567' }],
    // Inline definition
    function () {},
  ],
})
```

顺序规则在 callout 里，措辞很明确：

> **Note**: Modules are executed sequentially so the order is important. First, the modules defined in `nuxt.config.ts` are loaded. Then, modules found in the `modules/` directory are executed, and they load in alphabetical order.

也就是说生效顺序是**两段**：先是配置数组里的那一段（按数组顺序），再是 `modules/` 目录那一段（按文件名字母序）。目录那一段的默认位置由 `dir.modules` 决定，类型 `string`、默认 `"modules"`，说明句是 "The modules directory, each file in which will be auto-registered as a Nuxt module."。

## 两节对照着读的四处差别

| | `plugins` | `modules` |
| --- | --- | --- |
| 类型行 | `array` | `array` |
| 默认值行 | **无** | **无** |
| 条目形状 | 字符串 / `{ src, mode }` | 字符串 / `[名称, 选项对象]` 元组 / 内联函数 |
| 顺序规则 | callout 只说"要定制顺序才写进配置" | callout 明写"sequential，顺序重要"，并给出两段式 |
| 去重 | **明写 deduplicated by their src path** | **没有去重规则** |
| 目录自动注册 | `~/plugins` | `modules/` 目录（`dir.modules`，字母序） |
| See 指向 | `directory-structure/app/plugins` | 无 See（本节点到目录为止） |

两处"文档未说明"要按字面理解：**同一个模块字符串在 `modules` 里出现两次会怎样**，这一节没有去重句，不能借 `plugins` 那条推过来；**`mode` 能否写 `all`**，本节只出现过 `client` 与 `server`。

## 自查

- 两节都是 `Type: array`、**都没有默认值行**。
- 插件靠**后缀**或 `mode` 限定上下文，两种写法并存于同一示例，等价性无记载。
- 想改插件执行顺序才把它们写进配置；不写也在跑（目录自动注册）。
- **重复声明的插件按 src 路径去重**，所以"写了没生效"要先怀疑是不是路径不同（别名、扩展名、相对/绝对写法都算不同 src）。
- 模块顺序是两段式：配置的数组序 → `modules/` 目录的字母序。
- 模块条目可以带选项（元组第二元素），也可以直接内联一个函数。

## 下一篇

下一篇 584 看 `## extends`：这一节只有三行——多个来源、字符串或字符串数组、四种远程前缀——而真正的优先级阶梯写在图层那一页：**你的项目文件 > `~~/layers` 自动扫描的层（字母序，Z 高于 A）> `extends` 里的层（越靠前越高）**。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## plugins` 定性句、两种条目形状与 `.client`/`.server` 句、`Type: array`、`::callout` 三句、See、六条示例及其注释；`## modules` 定性句、三种条目形状句、解析规则句、`Type: array`、`::callout` 顺序句、四条示例；`### dir.modules` 说明句与默认 `"modules"`）
