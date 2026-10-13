---
url: /posts/692dddaaeb191f7734eb9a753777b420/
title: 097 - DevOnly：只在开发期保留的子树与它的生产替身
date: 2026-11-13T10:00:00+08:00
updated: 2026-11-13T10:00:00+08:00
author: cmdragon

summary:
  摘要：`<DevOnly>` 的官方定义只有一句——"Nuxt 提供 `<DevOnly>` 组件，只在开发期间渲染一个组件"，紧跟一句同样重要的结论：**"该内容不会被包含在生产构建中。"** 它对外只有一个插槽：`#fallback`，官方说明是"如果你确实需要在生产期间有一个替代物"。官方示例把这两件事放在一起写得很清楚：默认插槽里放 `<LazyDebugBar />`（注释："this component will only be rendered during development"），`#fallback` 里放一个空 div（注释："if you ever require to have a replacement during production"，且那个空 div 的注释写着 "empty div for flex.justify-between"，说明替身存在的理由常常是**布局不能塌**）；而在两处示例里，官方都塞了同一句提醒——"be sure to test these using `nuxt preview`"。这句话不是客套：开发模式下默认插槽永远成立，你根本看不到 `#fallback` 那一支，只有构建之后再预览才有机会验证它，而 `nuxt preview` 的官方定义正是"preview 命令启动一个服务器来预览你在运行 build 命令之后的 Nuxt 应用"（`nuxt start` 是同一个命令的另一个名字）。与 `<ClientOnly>` 的分界也要划清：前者按**环境**切（开发期 vs 生产构建），后者按**运行侧**切（服务端构建 vs 浏览器）。

categories:
  - 前端开发

tags:
  - nuxt4
  - devonly
  - 开发期
  - preview
  - 插槽

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


调试条、埋点面板、临时看数据的块——这些东西上线时不该存在。`<DevOnly>` 就是那把剪刀，但它有一个反直觉的地方：**它的生产替身，你在开发模式下永远看不到。**

## 官方只有两句话

| 官方原话 | 含义 |
| --- | --- |
| "Nuxt 提供 `<DevOnly>` 组件，只在开发期间渲染一个组件。" | 切的是**环境**：开发期 vs 生产 |
| "该内容不会被包含在生产构建中。" | 不是"运行时隐藏"，是**不进产物** |

第二条是关键。它意味着 `#fallback` 之外的那部分内容在生产包里根本不存在——不用担心它被下载、被查源码看到，也别指望在生产里靠它做开关。

## 唯一的插槽：`#fallback`

官方的插槽表就一行：**"`#fallback`：如果你确实需要在生产期间有一个替代物。"**

完整的官方示例把两个角色都写了注释：

```vue [app/pages/example.vue]
<template>
  <div>
    <Sidebar />
    <DevOnly>
      <!-- this component will only be rendered during development -->
      <LazyDebugBar />

      <!-- if you ever require to have a replacement during production -->
      <!-- be sure to test these using `nuxt preview` -->
      <template #fallback>
        <div><!-- empty div for flex.justify-between --></div>
      </template>
    </DevOnly>
  </div>
</template>
```

拆开读这段，有三层信息：

- **默认插槽放的是"只给开发看的东西"**，示例用了 `<LazyDebugBar />`——懒加载写法本身也说明它不该进生产包。
- **`#fallback` 放的是一个空 `<div>`**，而它的注释是 `empty div for flex.justify-between`。这句注释暴露了替身最常见的真实用途：**父容器是 flex 布局，元素突然消失会让整排位置跑偏**，所以留一个空壳把位置占住。
- 官方在同一个例子里两次写上 `be sure to test these using \`nuxt preview\``（插槽那一节的示例里又出现了一次）。

## 为什么替身必须用 `nuxt preview` 测

这是这一篇最实际的一条。原因不是命令偏好，而是两个组件的行为决定的：

| 你在哪儿看 | `<DevOnly>` 渲染什么 |
| --- | --- |
| `nuxt dev` | **永远渲染默认插槽**——此刻就是"开发期间"，`#fallback` 那一支没有任何机会被执行 |
| 构建后预览 | 生产形态，才会渲染 `#fallback` |

所以"我在开发模式下看过，占位没问题"这句话对 `#fallback` 是无效的——它从来没被渲染过。

而官方给的正是这条路径：`preview` 命令**启动一个服务器来预览你在运行 `build` 命令之后的 Nuxt 应用**（官方同时说明 `nuxt start` 是同一个命令的另一个名字）。也就是说要验这一支，顺序必须是 `nuxt build` → `nuxt preview`。

| 命令 | 官方定义 |
| --- | --- |
| `nuxt build` | 生产构建，产物是 `.output`（029、037 两篇） |
| `nuxt preview` | 启动一个服务器来预览 `build` 之后的应用；`nuxt start` 是它的另一个名字 |

预览端口同样可以指：`-p, --port`（默认链是 `NUXT_PORT || NITRO_PORT || PORT`）。另外这一节还有一个细节，官方写明 preview 会把自己记录在 `node_modules/.cache/nuxt/preview`，所以同一个项目的第二个 preview 实例会按 `--takeover` / `--no-takeover` 的规则处理已在跑的那个（039 展开过）。

## 与 `<ClientOnly>` 的分界

两个组件都"让一部分内容消失"，但切的维度不同，选错了就是白包：

| | `<DevOnly>` | `<ClientOnly>` |
| --- | --- | --- |
| 按什么切 | **环境**：开发期 / 生产构建 | **运行侧**：服务端构建 / 浏览器 |
| 消失的时机 | 生产构建，内容不进产物 | 服务端构建，默认插槽内容被 tree-shake 出服务端构建 |
| 替身机制 | `#fallback` 插槽 | `placeholder`/`fallback` prop 或 `#fallback` 插槽 |
| 典型用途 | 调试条、开发期才有的面板 | 依赖浏览器 API 的组件、请求相关内容 |
| 怎么验证替身 | **必须 `nuxt build` + `nuxt preview`** | dev 下刷新看首屏 HTML 是否含该块 |

两者还能叠加：需要"只在客户端、且只在开发期"的东西，就是 `<ClientOnly>` 外面套 `<DevOnly>` 的组合，各自的替身也各自负责。

## 该用哪个：三个具体判断

| 你的东西 | 选择 |
| --- | --- |
| 只给开发看的调试条/面板 | `<DevOnly>` 默认插槽；如果它在 flex 行里，按官方示例给一个空 `<div>` 占位 |
| 依赖 `window` 的第三方组件 | `<ClientOnly>`，不是 `<DevOnly>`——它上线路要照常显示 |
| 上线后仍然要显示、但内容取决于当前请求 | `<ClientOnly>`（096 那种兜底错误页写法） |
| 一段临时排查用的模板，忘了删也不该进包 | `<DevOnly>`——官方写明内容不会进生产构建 |

## 自查

1. `<DevOnly>` 按什么维度切？官方关于"内容去哪了"的原话是哪一句？
2. 它对外提供什么插槽？官方给的适用条件是什么？
3. 官方示例里那个空 `<div>` 的注释写了什么？这暴露了什么用途？
4. 为什么开发模式下永远验证不了 `#fallback`？正确的验证顺序是什么？
5. "依赖浏览器 API 的组件"该用 `<DevOnly>` 还是 `<ClientOnly>`？为什么？

## 下一篇

098 讲 `<Teleport>` 在 Nuxt 里的挂载点约定：`to` 能接受什么，为什么官方写"目前只对 `#teleports` 有 SSR 支持"，以及其他目标为什么要包一层 `<ClientOnly>`。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/dev-only`
  - 原文（标题与描述）："Render components only during development with the \<DevOnly> component."
  - 原文："Nuxt provides the `<DevOnly>` component to render a component only during development."；"The content will not be included in production builds."
  - 原文示例（文件标注 `app/pages/example.vue`）：`<!-- this component will only be rendered during development --> <LazyDebugBar />`；`<!-- if you ever require to have a replacement during production -->`；`<!-- be sure to test these using \`nuxt preview\` -->`；`<template #fallback><div><!-- empty div for flex.justify-between --></div></template>`
  - 原文（Slots）："`#fallback`: if you ever require to have a replacement during production."，其示例里再次出现 `<!-- be sure to test these using \`nuxt preview\` -->`
  - 原文 Source：`packages/nuxt/src/app/components/dev-only.ts`
- `https://nuxt.com/docs/4.x/api/commands/preview`
  - 原文："The `preview` command starts a server to preview your Nuxt application after running the `build` command. `nuxt start` is the same command under another name. When running your application in production refer to the Deployment section."
  - 原文（选项）："`-p, --port=<port>` | Port to listen on (default: `NUXT_PORT \|\| NITRO_PORT \|\| PORT`)"；"`--envName=<environment>` | The environment to use when resolving configuration overrides (default is `production` when building, and `development` when running the dev server)"
  - 原文（Taking over a running preview）："A preview records itself in `node_modules/.cache/nuxt/preview`, so a second `nuxt preview` for the same project deals with the one already running the way `nuxt dev` deals with a running dev server"
  - 原文："`--takeover` | Stop a preview server already running on this project and take its place"；"`--no-takeover` | Never stop a preview server already running on this project"
  - 原文："When the configured `server.builder` produces no server at all (a client-only build, for example), there is nothing to run: the static output is served directly instead, with unmatched paths falling back to the client entry so client-side routing works."
- `https://nuxt.com/docs/4.x/api/components/client-only`
  - 对照用（见 096）："The `<ClientOnly>` component is used for purposely rendering a component only on client side."；"The content of the default slot will be tree-shaken out of the server build."；"`#fallback`: specify a content to be rendered on the server and displayed until `<ClientOnly>` is mounted in the browser."
