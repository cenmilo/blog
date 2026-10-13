---
url: /posts/a8ba220f5a8c870fba34e79cfde648a9/
title: 562 - analyzeDir：分析报告写到哪、由谁读
date: 2027-04-17T10:00:00+08:00
updated: 2027-04-17T10:00:00+08:00
author: cmdragon

summary:
  摘要：`analyzeDir` 的说明是一句：**"The directory where Nuxt will store the generated files when running `nuxt analyze`."**，再补一句**"If a relative path is specified, it will be relative to your `rootDir`."**——类型 `string`，默认 **`"/<rootDir>/.nuxt/analyze"`**。它与另一处默认值是同一条链：`build.analyze`（以及 `webpack.analyze`）的默认 `filename` 写的是 **`"/<rootDir>/.nuxt/analyze/{name}.html"`**，其中 `{name}` 对应 `nuxt analyze --name` 的值、**不传时默认就是 `default`**。命令侧还有两条会影响你怎么"看到"这些文件：分析结果构建完会起本地服务展示，**除非传 `--no-serve` 或环境里设了 `CI`**；而且**分析时路由不会被预渲染**（要就加 `--prerender`）。这一节本身没有示例，也没说改这个目录会不会连带影响 `analyze.filename`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 打包分析
  - analyzeDir
  - 构建
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`nuxt analyze` 跑完，浏览器里看到了 treemap；但流水线里没有浏览器，你要的是那个 html 文件。

它在哪，由两个配置项决定，而其中一个是这一节。

## 这一节的全部记载

`## analyzeDir`：

> The directory where Nuxt will store the generated files when running `nuxt analyze`.
>
> If a relative path is specified, it will be relative to your `rootDir`.

- **类型**：`string`
- **默认**：`"/<rootDir>/.nuxt/analyze"`

两句各自的作用：**第一句把它绑到一条命令**（`nuxt analyze`，不是 `nuxt build`，也不是 `nuxt dev`）；**第二句给了相对路径的基准**——与 `srcDir`、`serverDir`、`typesDir` 一样按 `rootDir` 算。默认值里的 `<rootDir>` 是占位符，读作"项目根下的 `.nuxt/analyze`"。

## 它和 analyze.filename 是同一条链

配置参考里另一处默认值把它接上了：`build.analyze` 与 `webpack.analyze` 两处给的都是同一份 JSON——

```json
{
  "template": "treemap",
  "projectRoot": "/<rootDir>",
  "filename": "/<rootDir>/.nuxt/analyze/{name}.html"
}
```

三行串起来就是完整的落点：

| 来源 | 值 | 含义 |
| --- | --- | --- |
| `analyzeDir` 默认 | `/.nuxt/analyze`（相对 `rootDir`） | 生成文件的目录 |
| `build.analyze.filename` 默认 | `/.nuxt/analyze/{name}.html` | 具体报告文件名，`{name}` 是占位 |
| `nuxt analyze --name` 默认 | `default` | 决定了 `{name}` 的实值 |

**三者默认值互相对齐**（同一目录、带 `{name}`、不传参时是 `default`）。所以不传任何参数时，产物是 `.nuxt/analyze/default.html`；`npx nuxt analyze --name client` 之类就会换文件名。

**这一节没有说改 `analyzeDir` 会不会连带改 `filename`。** 两处各自给了默认值、彼此独立，稳妥做法是要挪位置就两处一起写，然后按文件名去实际找。

## 命令侧两条会影响"看不看得到"

`nuxt analyze` 那一节有两句正好落在这节的使用场景上：

> The `analyze` command builds Nuxt and analyzes the production bundle (experimental). The results are served on a local server when the build finishes, unless you pass `--no-serve` or `CI` is set in the environment.
>
> Routes are not prerendered while analyzing, since prerendering runs the built app and its output is not what is being measured. Pass `--prerender` if your analysis needs it.

两条推论：**在 CI 里它默认不起服务**（只生成文件），所以流水线要取那个 html 就得按默认路径去读，或自己传 `--name` 固定文件名；**分析时没有预渲染**，所以如果你在测"预渲染后的体积分布"，得显式加 `--prerender`，否则测的是另一套东西。

另外这条命令的定位词是 **experimental**——分析功能本身在文档里就标着实验性，**产物形状不保证稳定**。

## 为什么它落在 `.nuxt` 里

默认位置在 `.nuxt/analyze`，这一点有两个附带后果：

**一、`.nuxt` 是生成目录。** `nuxt cleanup` 的清除清单第一项就是"你的构建目录（默认 `.nuxt`，除非 `buildDir` 另有说明）"——**清理会把分析报告一起带走**，这与"报告文件是临时产物"的定位一致。

**二、它会跟着 `buildDir` 一起改位置吗？** 这一节没说。**但 `analyze.filename` 的默认值里写的是字面 `.nuxt/analyze/`**，也就是说改 `buildDir`（比如换成 `nuxt-build`）之后，两处默认值就不再对齐。要确认，最短的做法是改完 `buildDir` 跑一次 `npx nuxt analyze --no-serve`，再在整个项目里找那个 html 出现在哪个目录。

（上一条是本系列给的验证方法，官方在这两节都没给迁移说明。）

## 自查

- `analyzeDir` 是 `string`、默认 `"/<rootDir>/.nuxt/analyze"`，只服务 `nuxt analyze` 这一条命令；相对路径按 `rootDir` 算。
- 它与 `build.analyze`（＝`webpack.analyze`）的默认 `filename: "/<rootDir>/.nuxt/analyze/{name}.html"` 对齐，`{name}` 由 `--name` 决定，**不传时是 `default`**。
- CI 或传 `--no-serve` 时不起本地服务，只生成文件；分析时不预渲染，需要就加 `--prerender`。
- 这条命令被官方标注为 experimental，输出形状不保证稳定。
- 报告在 `.nuxt` 下，会被 `nuxt cleanup` 一并清除；改 `analyzeDir` 或 `buildDir` 是否连带影响 `filename`，两处文档都没写，要实测。

## 下一篇

530-599 继续。下一篇 563 读 `nitro` 节的上半：它在配置参考里只有两句加两个子项，真正的可配范围在 Nitro 自己的文档里——这一节是"边界"而不是"清单"。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## analyzeDir` 整节：两句说明、Type `string`、Default `"/<rootDir>/.nuxt/analyze"`；`### build.analyze` 与 `### webpack.analyze` 的同一份三项默认 JSON 含 `filename` 里的 `{name}.html`；`## buildDir` 默认 `"/<rootDir>/.nuxt"` 与其隐藏目录说明）
- https://nuxt.com/docs/4.x/api/commands/analyze（命令定性、experimental 标注、`--no-serve` 与设了 `CI` 时不起服务、分析时不预渲染需 `--prerender`、`--name` 默认 `default`）
- https://nuxt.com/docs/4.x/api/commands/cleanup（清除列表第一项"构建目录，默认 `.nuxt` 除非 `buildDir` 另有说明"）
