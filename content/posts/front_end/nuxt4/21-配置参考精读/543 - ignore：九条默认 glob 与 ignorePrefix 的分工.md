---
url: /posts/6e34d5b8c20e14846dbf467bffd4335a/
title: 543 - ignore：九条默认 glob 与 ignorePrefix 的分工
date: 2027-04-11T09:00:00+08:00
updated: 2027-04-11T09:00:00+08:00
author: cmdragon

summary:
  摘要：`ignore` 的定位是**"More customizable than `ignorePrefix`"**——凡是匹配这个数组里 glob 模式的文件，在构建时都会被忽略。官方给的默认值有九条 glob，覆盖三类东西：测试与故事文件（`**/*.stories.{js,cts,mts,ts,jsx,tsx}`、`**/*.{spec,test}.{js,cts,mts,ts,jsx,tsx}`）、类型声明（`**/*.d.{cts,mts,ts}`、`**/*.d.vue.{cts,mts,ts}`）、工具与缓存目录（`**/.{pnpm-store,vercel,netlify,output,git,cache,data}`、`.nuxt/analyze`、`.nuxt`），加上两条特殊形状：**`**/*.sock`**（socket 文件）与 **`**/-*.*`**（以 `-` 开头的文件——正是 `ignorePrefix` 默认值那条规则的 glob 版本）。所以两个键不是二选一：`ignore` 是模式数组，`ignorePrefix` 是"某个目录里文件名前缀"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建
  - ignore
  - glob
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

把 `Button.spec.ts` 放进 `app/pages/` 目录附近试一下，路由表里会多出一条 `/button-spec`。

测试文件没被排除干净，是因为你改过 `ignore` 而不知道它的默认值里有哪九条。

## 定义句与九条默认值

`## ignore` 的说明只有一句，但那句同时交代了它和另一个键的关系：

> More customizable than `ignorePrefix`: all files matching glob patterns specified inside the `ignore` array will be ignored in building.

- **类型**：`array`
- **默认**（九项，逐字照录）：

```json
[
  "**/*.stories.{js,cts,mts,ts,jsx,tsx}",
  "**/*.{spec,test}.{js,cts,mts,ts,jsx,tsx}",
  "**/*.d.{cts,mts,ts}",
  "**/*.d.vue.{cts,mts,ts}",
  "**/.{pnpm-store,vercel,netlify,output,git,cache,data}",
  "**/*.sock",
  ".nuxt/analyze",
  ".nuxt",
  "**/-*.*"
]
```

按用途分组看：

| 组 | 条目 | 挡住什么 |
| --- | --- | --- |
| 故事与测试 | `.stories.*`、`{spec,test}.*` | Storybook 文件、单测与 e2e 文件被当成页面或组件 |
| 类型声明 | `*.d.{cts,mts,ts}`、`*.d.vue.{cts,mts,ts}` | `.d.ts` 与 `.d.vue.ts` 这类只给类型用的文件 |
| 工具与缓存目录 | `**/.{pnpm-store,vercel,netlify,output,git,cache,data}` | 六点点目录被扫描 |
| 生成目录 | `.nuxt/analyze`、`.nuxt` | 分析与构建产物自身 |
| 特殊形状 | `**/*.sock`、`**/-*.*` | socket 文件；以 `-` 开头的文件 |

三点值得停一下：

**第一，展开写法是 brace 形式。** `{js,cts,mts,ts,jsx,tsx}` 这种花括号列举出现在前四条里，意思是"这六种扩展名各匹配一次"，不是正则。

**第二，第二条覆盖测试文件。** `**/*.{spec,test}.{js,cts,mts,ts,jsx,tsx}` 一条同时管 `.spec.*` 与 `.test.*`——**这就是为什么默认情况下测试文件不会被注册成页面**（测试文档给的示例文件名正是 `.test.ts` 与 `.spec.ts` 两种）。**注意它不含 `.vue`**：测试文件的扩展名集合是那六种 JS/TS 形态。

**第三，最后一条与前缀键重复。** `**/-*.*` 匹配的正是"以 `-` 开头的文件"，也就是 `ignorePrefix` 默认值 `"-"` 的 glob 版。**两处默认表达同一条规则**，改的时候不要只改一处就以为规则换掉了。

## 两个键的分工

同页另一节给的正是 `ignorePrefix` 的机制（下一篇细读），这里先看两者的适用差别：

| 键 | 官方措辞 | 生效范围 |
| --- | --- | --- |
| `ignore` | all files matching **glob patterns** inside the array will be ignored in building | 任意路径，按模式匹配 |
| `ignorePrefix` | 文件名以指定前缀开头的文件会被忽略 | **点名四个目录**：`app/pages/`、`app/layouts/`、`app/middleware/`、`public/` |

所以判断标准是这样的：**要按路径形态排除（某些后缀、某些目录）用 `ignore`；要按"文件名前缀"排除、而且范围就是那四个目录，用 `ignorePrefix`。** 前者的默认值有九条，后者默认只有一个字符 `-`。

## 覆盖默认值的正确姿势

这是这节最容易出事的地方：**这一节没有写数组是追加还是替换。** 文档给的形态是 `- **Type**: array` 加一份"Default"JSON，配置项赋数组值时是完整替换还是并入默认，官方在本节没有一句话说明。

所以最稳的写法是**把要保留的默认项一起写进去**，例如"想多排除一类，但不丢测试文件规则"：

```ts
export default defineNuxtConfig({
  ignore: [
    '**/*.mdx',
    '**/*.{spec,test}.{js,cts,mts,ts,jsx,tsx}',
    '**/-*.*',
  ],
})
```

**上面这段是本系列给的写法示例，官方在 `## ignore` 这一节没有给任何 Example。** 三条里第一条是你新加的，后两条是从默认值里抄回来保住的。

要确认替换语义，最短的办法是做一次对照：只写一条自定义 glob、构建后看 `.spec.ts` 文件是否仍被排除。如果注册出了多余路由，说明默认值被整体替换了。

## 自查

- `ignore` 是 `array`、默认九条 glob，分五组（故事与测试、类型声明、工具点目录、`.nuxt` 与 analyze、`*.sock` 与 `-*.*`）。
- 官方第一句就把它与 `ignorePrefix` 的关系写明：**`ignore` 比它更可定制**。
- 第二条 `{spec,test}` 覆盖那六种 JS/TS 扩展名、**不含 `.vue`**，这是测试文件默认不成页面的原因。
- 最后一条 `**/-*.*` 与 `ignorePrefix` 的默认 `"-"` 表达同一条规则，两处都要看。
- 数组是追加还是替换本节没写；自定义时把要保留的默认项一起列上，并用一次构建对照验证。

## 下一篇

下一篇 544 读 `ignorePrefix` 与 `ignoreOptions`：前者点名四个目录、只认文件名前缀，后者把选项直接交给 node-ignore（官方示例是关掉大小写不敏感）。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## ignore` 整节：定义句与"More customizable than `ignorePrefix`"、Type `array`、九项默认 glob 全文；相邻的 `## ignorePrefix` 四个目录与默认 `"-"`、`## ignoreOptions` 的 node-ignore 定位）
- https://nuxt.com/docs/4.x/getting-started/testing（官方测试示例文件名使用 `.test.ts` 与 `.spec.ts` 两种后缀，用于对照第二条默认 glob 的覆盖面）
