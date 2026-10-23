---
url: /posts/bbcd4f0ed79c054258641988555e7aab/
title: 104 - spa-loading-template：关掉 SSR 后白屏期的那份 HTML
date: 2026-11-15T11:00:00+08:00
updated: 2026-11-15T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`spaLoadingTemplate` 官方定义为"布尔值，或一个 HTML 文件的路径，其内容将被插入到**任何以 `ssr: false` 渲染的 HTML 页面**中"，默认值是 `null`，而三种取值的行为各不一样：**未设置时，它会使用你的某一层里的 `~/spa-loading-template.html` 文件（如果它存在）；设为 `false` 时不会加载任何 SPA 加载指示器；设为 `true` 时 Nuxt 会在你的某一层里查找 `~/spa-loading-template.html`，找不到就使用一个 Nuxt 默认图像**。渲染概念页还单独给了一条应当照做的提醒：**"如果你确实使用了 `ssr: false`，你也应当在 `~/spa-loading-template.html` 放置一个 HTML 文件，写入你希望用来渲染加载屏的 HTML；它会一直渲染到你的应用完成水合。"** 官方为这一节附上一整段可直接用的加载屏示例（`<div class="loader"></div>` 配一段固定定位、50% 位移、`border-radius: 50%`、`animation: loader 400ms linear infinite` 的 CSS），并推荐了两个找 spinner 的来源（SpinKit 与 SVG Spinners）。要划清的界线是：这份模板**只在 `ssr: false` 的页面里出现**，它比 `app.vue` 更早、且不属于 Vue 组件树；它和开发服务器的 `devServer.loadingTemplate`（官方给的类型是 function）不是同一个东西；也和 `<ClientOnly>` 的 `#fallback`（组件级占位、显示到该组件在浏览器挂载为止）不同一层。另外那句"在某一层里"是有实际含义的——**图层也能提供这份模板**，这与"每个 layer 必须有 `nuxt.config.ts`"的图层机制是同一套解析路径。

categories:
  - 前端开发

tags:
  - nuxt4
  - 加载屏
  - ssr
  - 水合
  - 配置

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


关掉服务端渲染之后，用户点开链接看到的是白屏——JS 还在下载。这一段空白期不是只能干等，Nuxt 给了一个专门的位置放"先显示点什么"。这一篇讲它怎么用、什么时候不生效。

## 官方定义与三种取值

`spaLoadingTemplate`：**布尔值，或一个 HTML 文件的路径，其内容将被插入到任何以 `ssr: false` 渲染的 HTML 页面中。** 默认值是 `null`。

三种情况官方写得很细：

| 取值 | 官方描述的行为 |
| --- | --- |
| **未设置**（默认 `null`） | 会**使用你的某一层（layer）里的 `~/spa-loading-template.html` 文件，如果它存在** |
| `false` | **不会加载任何 SPA 加载指示器** |
| `true` | Nuxt 会在你的某一层里查找 `~/spa-loading-template.html`，**找不到就使用一个 Nuxt 默认图像** |

读这张表要抓住三件事：

1. **"未设置"≠"关闭"。** 默认值下，只要文件存在就会被用；要真的关掉才写 `false`。
2. **`true` 是兜底开关。** 它保证即使没有文件也有一张默认图，用户不会面对纯白屏。
3. **"任何以 `ssr: false` 渲染的 HTML 页面"** ——这句话就是它的生效范围：**服务端渲染开着的时候，这份模板根本不会出现在你的页面上。**

## 文件放在哪儿

`~/spa-loading-template.html`。这里的 `~` 是别名，官方给的默认值把它指向 `"/<rootDir>/app"`，也就是应用目录（`srcDir`，Nuxt 4 默认是 `app`）。所以这个文件的实际位置是 **`app/spa-loading-template.html`**。

如果你配了自定义 `srcDir`，它跟着新的 `srcDir` 走，这一点和同一目录里的 `app.config.ts` 是同一个规则。

另一句官方措辞也值得单独看：**"它会在你的某一层里查找"** ——这意味着**图层也可以带这份模板**。这跟图层机制本身是一致的（每个 layer 都必须有 `nuxt.config.ts`），所以一个共享外壳的图层组可以把加载屏一起放进层里，项目不用各自复制一份。

## 应当配它的条件

渲染概念页给的是"应当"级别的提醒，原文：

> 如果你确实使用了 `ssr: false`，你也应当在 `~/spa-loading-template.html` 放置一个 HTML 文件，写入你希望用来渲染加载屏的 HTML；**它会一直渲染到你的应用完成水合。**

这一句同时给出了显示时长：**从首屏 HTML 出现，到水合完成为止**。它不是"取数期间"的 loading，而是整个 Vue 应用还没接手的那段。

配套的开关就是那一行配置：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  ssr: false,
})
```

## 官方给的加载屏示例

官方在这一节直接贴了一整份可用文件（`~/spa-loading-template.html`），注释标明了样式来源：

```html
<!-- https://github.com/barelyhuman/snips/blob/dev/pages/css-loader.md -->
<div class="loader"></div>
<style>
.loader {
  display: block;
  position: fixed;
  z-index: 1031;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 18px;
  height: 18px;
  box-sizing: border-box;
  border: solid 2px transparent;
  border-top-color: #000;
  border-left-color: #000;
  border-bottom-color: #efefef;
  border-right-color: #efefef;
  border-radius: 50%;
  -webkit-animation: loader 400ms linear infinite;
  animation: loader 400ms linear infinite;
}

@-webkit-keyframes loader {
  0% {
    -webkit-transform: translate(-50%, -50%) rotate(0deg);
  }
  100% {
    -webkit-transform: translate(-50%, -50%) rotate(360deg);
  }
}
@keyframes loader {
  0% {
    transform: translate(-50%, -50%) rotate(0deg);
  }
  100% {
    transform: translate(-50%, -50%) rotate(360deg);
  }
}
</style>
```

从这份示例能读出三个必要约束：

- **它是一个 `<div>` 加一段 `<style>`**——纯 HTML 加内联样式，没有组件、没有脚本、没有外部资源。这段时间里 Vue 还没运行，任何依赖框架或网络的写法都不可靠。
- 样式是**自包含**的：`position: fixed` + `top/left: 50%` + `transform: translate(-50%, -50%)` 让它脱离父级布局居中，`z-index: 1031` 保证浮在最上；旋转由 `animation: loader 400ms linear infinite` 驱动，配套的 `@keyframes` 也在这同一份文件里。
- `-webkit-` 前缀与标准写法**成对**给出（`-webkit-animation` / `animation`、两套 `@keyframes`）——这段时间还没有构建流程替你处理兼容，所以示例把前缀写全。

官方另外推荐了两个找 spinner 的地方：**SpinKit** 与 **SVG Spinners**。

## 三个同名"loading"不是一回事

这是最容易配错的一组：

| 名字 | 位置与形态 | 生效条件 | 消失时机 |
| --- | --- | --- | --- |
| `spaLoadingTemplate` / `app/spa-loading-template.html` | `nuxt.config` 里的布尔或路径；文件是**纯 HTML** | **只有以 `ssr: false` 渲染的页面** | 应用完成水合 |
| `devServer.loadingTemplate` | `devServer` 节下的一个子项，官方给的类型是 `function` | 本地开发服务器 | 由开发服务器决定 |
| `<ClientOnly>` 的 `#fallback` | 组件的插槽，可以是节点 | 只在服务端渲染那一次显示 | `<ClientOnly>` 在浏览器挂载为止 |
| `<NuxtLoadingIndicator>` | 组件，放在 `app.vue` 或布局里、与页面出口并列 | 应用已运行，**导航之间** | 加载结束（`throttle` 会吞掉快于 200 毫秒的起止） |

四条按时间轴排一下就清楚了：**spa 加载屏（水合前）→ 应用运行 → 换页时的进度条 → 单个组件块的占位**。

顺带一句：`ssr: false` 时的产物形态与静态兜底文件（`index.html` + `200.html` + `404.html`）是另一件事（103 讲过），别把"关掉 SSR"和"生成静态站"混成一个配置。

## 自查

1. `spaLoadingTemplate` 三种取值分别发生什么？默认那一档是"关闭"吗？
2. 这份模板插进什么样的页面？服务端渲染开着时它会出现在哪里？
3. `~/spa-loading-template.html` 的实际路径是什么？自定义 `srcDir` 时它跟谁走？
4. 官方示例为什么把 `-webkit-` 前缀写全、为什么不用脚本？
5. `spaLoadingTemplate`、`devServer.loadingTemplate`、`<ClientOnly>` 的 `#fallback`、`<NuxtLoadingIndicator>` 四者按时间轴各站在哪一段？

## 下一篇

105 讲读官方文档时会撞上的两种符号：`#imports`、`#server`、`#build` 这类以 `#` 开头的导入别名分别指向哪里，以及文档里那些 `::note`、`::code-group`、`::read-more` 是什么语法、为什么不能直接粘进 `.vue` 文件。
