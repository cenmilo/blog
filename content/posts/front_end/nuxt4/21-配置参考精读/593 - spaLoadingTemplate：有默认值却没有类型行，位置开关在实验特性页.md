---
url: /posts/47724474a20ef0c8f9c50a7ab0120ee9/
title: spaLoadingTemplate：有默认值却没有类型行，位置开关在实验特性页
date: 2027-04-27T11:00:00+08:00
updated: 2027-04-27T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## spaLoadingTemplate` 的类型说明写在定性句里——**"Boolean or a path to an HTML file with the contents of which will be inserted into any HTML page rendered with `ssr: false`."**——而这一节**有 `**Default:** null` 却没有 `**Type**` 行**。三条分支（不设／`false`／`true`）在原文里被挤成同一段。要决定这块加载屏渲染在哪，得翻到实验特性那一页的 `spaLoadingTemplateLocation`：`within` 渲染在 `<div id="__nuxt">` 内部，`body` 渲染在它旁边，官方说后者**"avoids a white flash when hydrating a client-only page"**。还有两个键在管这个元素本身——`app.spaLoaderTag`（默认 `"div"`）与 `app.spaLoaderAttrs.id`（默认 `"__nuxt-loader"`），**这四处分在两节，文档从没放在一起讲过。**

categories:
  - 前端开发

tags:
  - nuxt4
  - 加载屏
  - 客户端渲染
  - spa-loading-template
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

关掉 SSR 之后，白屏会一直持续到 JavaScript 下载、解析、水合完成。官方给的那块占位加载屏由四个键控制，它们分散在三处，文档从来没把它们串起来讲。

## 这一节的记载

`## spaLoadingTemplate` 的定性句自己就是类型说明：

> Boolean or a path to an HTML file with the contents of which will be inserted into any HTML page rendered with `ssr: false`.

接着是三条分支，**原文把它们连在一段里**（照抄格式）：

> - If it is unset, it will use `~/spa-loading-template.html` file in one of your layers, if it exists. - If it is false, no SPA loading indicator will be loaded. - If true, Nuxt will look for `~/spa-loading-template.html` file in one of your layers, or a default Nuxt image will be used.

然后是一句素材指引：**"Some good sources for spinners are [SpinKit](https://github.com/tobiasahlin/SpinKit) or [SVG Spinners](https://icones.js.org/collection/svg-spinners)."**

- **没有 `**Type**` 行。**
- **Default:** `null`

这一节的格式缺陷值得点出来：**定性句承担了类型说明，但类型行是缺的**（配置参考 60 个顶层节里，这一节是唯一"有默认值却没有类型行"的）。同时三条 `if` 挤成一段，读的时候容易漏掉中间那条 `false`。

三份语义要分开记：

| 取值 | 官方说法 |
| --- | --- |
| 不设 | 会用你某个图层里的 `~/spa-loading-template.html`（**如果它存在**） |
| `false` | 不加载任何 SPA 加载指示 |
| `true` | 找图层里的 `~/spa-loading-template.html`，**找不到就用 Nuxt 默认图** |

生效前提写在定性句末尾：**只进入"以 `ssr: false` 渲染的那些 HTML 页面"**。渲染概念页那句建议也对应这个前提——如果你要用 `ssr: false`，就应该在 `~/spa-loading-template.html` 放一份你想要的加载屏 HTML，它会一直渲染到应用完成水合。

## 位置开关在实验特性那一页

要决定这块 HTML 渲染在哪儿，配置参考这一节没有答案；`spaLoadingTemplateLocation` 是**实验特性**，条目在实验特性那一页。

它的开头两句：

> When rendering a client-only page (with `ssr: false`), we optionally render a loading screen (from `~/spa-loading-template.html`).
>
> It can be set to `within`, which will render it like this:

```html
<div id="__nuxt">
  <!-- spa loading template -->
</div>
```

> Alternatively, you can render the template alongside the Nuxt app root by setting it to `body`:

```html
<div id="__nuxt"></div>
<!-- spa loading template -->
```

最后一句是这个开关的全部理由：**"This avoids a white flash when hydrating a client-only page."**

要标注清楚三条空白：这一节**没有 `**Type**` 行、没有 `**Default**` 行、也没有那份常见的 `experimental: { … }` 配置示例**——取值只有 `within` 与 `body` 两个是从两句正文里读出来的，默认是哪一个没有写。而 `## experimental` 那一节是空壳（只有一条 read-more），所以想从配置页查这个开关的形状是查不到的。

顺带一提，`experimental` 那 57 个小节里只有 17 条用了固定句式写明默认状态，**这一条不属于其中任何一条**。

## 那两个管元素本身的键在哪

加载屏那个元素长什么样，由 `## app` 下面两个子项决定：

| 键 | 说明句 | Type | Default |
| --- | --- | --- | --- |
| `app.spaLoaderTag` | "Customize Nuxt SpaLoader element tag." | `string` | `"div"` |
| `app.spaLoaderAttrs` | "Customize Nuxt SPA loading template element attributes." | `object` | `{"id": "__nuxt-loader"}` |
| `app.spaLoaderAttrs.id` | **无说明句** | `string` | `"__nuxt-loader"` |

于是四件事各有归属：**放什么内容**在 `spaLoadingTemplate`、**渲染在哪**在 `experimental.spaLoadingTemplateLocation`、**用什么标签**在 `app.spaLoaderTag`、**这个标签带什么属性**在 `app.spaLoaderAttrs`。

文档没有把这条链条写在一起，也**没有说明这四者之间的优先级或配合规则**——比如 `spaLoadingTemplateLocation` 设成 `within` 时那个 `id="__nuxt-loader"` 的元素是插在 `#__nuxt` 里面（即与根元素并列于同一 div），这在两份记载里都没有图示。

## 自查

- `## spaLoadingTemplate`：**类型说明在定性句里、没有 Type 行**，有 `Default: null`，三条分支在原文挤成一段。
- 生效范围只限"以 `ssr: false` 渲染的页面"；`false` 完全不加载，`true` 会回落到默认图。
- 文件位置写的是 `~/spa-loading-template.html`，并且是**在你的某个图层里查找**——多层项目里就近的那层由图层优先级决定，本节没有复述规则。
- 位置开关 `spaLoadingTemplateLocation` 在实验特性页，取值 `within`／`body`，**无类型行、无默认值、无配置示例**；选 `body` 的动机是避免水合白屏。
- 元素本体由 `app.spaLoaderTag`（`"div"`）与 `app.spaLoaderAttrs`（`{"id":"__nuxt-loader"}`）管，**四者如何配合文档未串讲**。

## 下一篇

下一篇 594 看 `## appId`：定性句只有半句 **"For multi-app projects, the unique id of the Nuxt application."**，默认值 `nuxt-app` **在原文里写了两遍**（正文一句 "Defaults to `nuxt-app`." ＋ 一行 `**Default:** \`"nuxt-app"\``），而这一键到底被用在哪里，**整套 4.x 文档没有一处正面说明。**

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## spaLoadingTemplate` 定性句、三条分支原文、素材指引句、`Default: null` 与无 Type 行的事实；`### spaLoaderAttrs` 与 `#### id`、`### spaLoaderTag` 的说明句／类型／默认值）
- https://nuxt.com/docs/4.x/guide/going-further/experimental-features（`## spaLoadingTemplateLocation` 两句定性与 `within`／`body` 两份 HTML 图示、"This avoids a white flash when hydrating a client-only page."）
- https://nuxt.com/docs/4.x/guide/concepts/rendering（`ssr: false` 时应在 `~/spa-loading-template.html` 放加载屏 HTML、它会渲染到应用水合完成的那句 `::note`）
