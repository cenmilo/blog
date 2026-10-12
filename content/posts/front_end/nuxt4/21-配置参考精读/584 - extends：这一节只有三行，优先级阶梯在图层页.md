---
url: /posts/915e9930e1d9e9d8f9297698c42aee8d/
title: extends：这一节只有三行，优先级阶梯在图层页
date: 2027-04-24T11:00:00+08:00
updated: 2027-04-24T11:00:00+08:00
author: cmdragon

summary:
  摘要：`## extends` 的记载是三行加两条 See：**没有 `**Type**` 行、没有 `**Default**` 行**。三行说的是"从多个本地或远程源扩展项目"、"值是一个字符串或字符串数组，指向源目录或配置路径，**相对当前配置**"、"可以用 `github:`、`gh:`、`gitlab:`、`bitbucket:`"。真正决定行为的东西都在图层那一页：**优先级三级阶梯**（项目文件 > `~~/layers` 自动扫描层，字母序且 Z 高于 A > `extends` 数组，越靠前越高）、层里哪些目录会被自动扫描、`$meta.name` 生成的 `#layers/*` 别名、相对路径的两个坑，以及 v4.3 起"把模块的配置键设 `false` 就能关掉它"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 图层
  - 配置继承
  - 优先级
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一个团队把公共的组件、布局和 `app.config` 抽成一层，另一个项目里只写一行 `extends: ['../base']` 就都拿到了。接下来就撞上那个最要命的问题：**两层都定义了同一个组件，谁生效？** 这一节回答不了，它只有三行。

## 这一节的全部记载

`## extends`：

> Extend project from multiple local or remote sources.
>
> Value should be either a string or array of strings pointing to source directories or config path relative to current config.
> You can use `github:`, `gh:` `gitlab:` or `bitbucket:`

两条 See 分别指向 c12 的 "extending config layers" 文档和 giget 的文档。**类型行与默认值行都没有**——也就是说数组元素的确切类型、以及不写 `extends` 时是不是空数组，本节没写。

那句 "relative to current config" 的口径要注意：图层页后面又说明，**层自己的 `nuxt.config` 里的相对路径实际是相对用户项目解析的**（嵌套 `extends` 除外）。两句合起来读，结论是配置里写路径时要按"谁的配置"来判断基准，见下面那个坑。

远程前缀这一行列的是 `github:`、`gh:`、`gitlab:`、`bitbucket:`（原文 `gh:` 后面漏了逗号），**没列 `https://`**；隔壁 `## theme` 那节的列表里反而有 `https://`。两节的差异留到下一篇逐行对照。

## 图层页给的形状：一个数组里混三种源

图层页 "Basic Example" 给的是最小形状 `extends: ['./base']`，并配了一份 `base/nuxt.config.ts`（里面写的是 `app.head`）。稍长的示例把三种源放在同一个数组里：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    // Local layer outside the project
    '../base',
    // NPM package
    '@my-themes/awesome',
    // Remote repository
    'github:my-themes/awesome#v1',
  ],
})
```

**`#v1` 是远程源里唯一出现过的版本锁定写法**（指 git ref），文档没有展开它能用哪些 ref 形式。

## 优先级：三级阶梯加两条排序细节

图层页 Layer Priority 一节的顺序，从高到低是三条：

1. **你自己的项目文件**（永远最高）
2. **`~~/layers` 目录里自动扫描到的层**——按字母序排，**Z 的优先级高于 A**
3. **`extends` 配置里的层**——**靠前的优先级高于靠后的**

上面那个数组的例子，如果项目里同时存在 `~~/layers/custom`，官方给的完整排序是：项目文件 > `~~/layers/custom` > `../base` > `@my-themes/awesome` > `github:my-themes/awesome#v1`，并总结成两句："your project files will override any layer" 与 "`~~/layers/custom` will override anything in `extends`"。

想控制自动扫描层的顺序，官方给的技巧是**加数字前缀**：`~/layers/1.z-layer`、`~/layers/2.a-layer`，这样 `2.a-layer` 高于 `1.z-layer`。

用途分工也写了两句：`extends` 用于**外部依赖**（npm 包、远程仓库）或项目目录之外的层；`~~/layers` 目录用于**项目内的本地层**。

Kit 侧还有一个读这个顺序的入口：`getLayerDirectories()`，官方对它的两条注记是"**数组里靠前的优先级更高、会覆盖靠后的**"和"**用户项目是数组第一项**"。

## 一层里哪些东西会被带进来

图层页列了十项自动扫描的内容，全部标着 "Extend the default …"：

`app/components/*`、`app/composables/*`、`app/layouts/*`、`app/middleware/*`、`app/pages/*`、`app/plugins/*`、`app/utils/*`、`app/app.config.ts`、`server/*`、`nuxt.config.ts`。

**这十项解释了"为什么我没写 extends 之外的任何配置，组件和布局就都出现了"**——包括那两份容易忘的：`app/app.config.ts` 与 `nuxt.config.ts`（合并顺序由 defu 负责，见下一节）。

## 别名与两个路径坑

**别名的来源有两种。** 自动扫描的层自动得到别名，例如 `~~/layers/test` 可以用 `#layers/test` 访问；其它层要自己声明，办法是在**该层的配置**里写 `$meta`：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  $meta: {
    name: 'example',
  },
})
```

这会产出指向该层的 `#layers/example`。要注意：**`$meta` 这个键在配置参考里没有条目**，它只在这一处出现过。

**坑一是全局别名。** 层里的组件或组合式函数如果用具全局别名（`~/`、`@/`）导入，**这些别名相对用户项目的路径解析**，而不是相对层自己。官方给的绕法是改用相对路径，或用上面那种命名层别名。

**坑二是层配置里的相对路径。** 层 `nuxt.config` 里的相对路径（嵌套 `extends` 除外）同样相对用户项目解析，官方给的绕法是写完整解析后的路径：

```ts [nuxt.config.ts]
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const currentDir = dirname(fileURLToPath(import.meta.url))

export default defineNuxtConfig({
  css: [
    join(currentDir, './app/assets/main.css'),
  ],
})
```

## 关掉层带来的模块（v4.3）

图层页这一节标了 `v4.3`：把某个模块的**配置键**设成 `false` 就能禁用它。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: ['./base-layer'],
  // Disable modules from the layer by setting their config key to false
  image: false, // Disables @nuxt/image
  pinia: false, // Disables @pinia/nuxt
})
```

`::note` 说得很关键：**配置键由各模块自己定义**，官方举了 `image`（`@nuxt/image`）、`pinia`（`@pinia/nuxt`）、`content`（`@nuxt/content`）三个例子，其余要查对应模块文档。`::tip` 补了一句适用范围与副作用：**这招也能关自己项目里的模块，`setup` 函数不会运行，但类型仍然会生成。**

官方列的适用场景三条：层里有你用不到的模块、你想换成另一种实现、某些环境要禁掉统计类模块。

## 这套机制由谁实现

Going Deeper 一节把归属写清楚了：**配置加载与 `extends` 支持由 unjs/c12 负责，合并用 unjs/defu，远程 git 源靠 unjs/giget**，并让你去查它们的文档与源码。这一节还挂了一个 GitHub issue（nuxt/nuxt **#13367**）说明图层支持的改进仍在进行。

换句话说，`extends` 的合并语义（深合并、数组追加还是覆盖）**不在 Nuxt 文档里定义**，而是在 c12/defu 那一侧。想知道某个具体键怎么合，本节给不了答案。

## 自查

- 配置节 `## extends`：**三行 ＋ 两条 See，无 Type、无 Default**；远程前缀列了四个，**没有 `https://`**。
- 优先级阶梯在图层页：项目文件 > `~~/layers`（字母序，Z 高于 A）> `extends`（靠前高于靠后），可用数字前缀控制扫描层顺序。
- 一层被带进来的是**十项**（含 `app.config.ts` 与 `nuxt.config.ts`）。
- 别名：扫描层自动得到 `#layers/<目录名>`；其它层靠 `$meta.name`（**该键在配置参考里无条目**）。
- 两个路径坑：层里的 `~/`/`@/` 与层配置里的相对路径都相对**用户项目**解析；绕法分别是相对路径/命名别名与写全解析路径。
- 关层里的模块：v4.3 起把模块配置键设 `false`，`setup` 不跑但类型仍生成；键名由模块自定义。
- 合并语义属于 c12 ＋ defu ＋ giget，Nuxt 文档不定义。

## 下一篇

下一篇 585 把 `## theme` 与 `## extends` 逐行对照：两节的说明句几乎同形，差别只有三处——**`theme` 给了 `Type: string` 而 `extends` 没有类型行、`theme` 的前缀列表里有 `https://` 而没有 `gh:`、`theme` 的 See 一条都没有**；而整套 4.x 里**没有任何一处写过 `theme:` 的示例**，图层那一页也通篇只讲 `extends`。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## extends` 三句原文与两条 See；`## theme` 相邻节；本批同时记录 `## extends` 没有 Type 行、没有 Default 行）
- https://nuxt.com/docs/4.x/guide/going-further/layers（十项自动扫描内容、Basic Example、Layer Priority 三条阶梯与五行排序例、字母序 Z 高于 A 那句、数字前缀 `::tip`、When to Use Each、`#layers/*` 别名与 `$meta.name` 示例、Relative Paths and Aliases 两个坑与 `fileURLToPath` 绕法、Disabling Modules from Layers（v4.3）三条场景与两条 note/tip、getLayerDirectories 两条注记、Going Deeper 的 c12/defu/giget 归属与 issue #13367）
