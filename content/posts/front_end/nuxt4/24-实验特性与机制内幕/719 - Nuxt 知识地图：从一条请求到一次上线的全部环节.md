---
url: /posts/eac5382794c19f46f86900c6314f4531/
title: 719 - Nuxt 知识地图：从一条请求到一次上线的全部环节
date: 2027-06-08T11:00:00+08:00
updated: 2027-06-08T11:00:00+08:00
author: cmdragon

summary:
  摘要：把 Nuxt 的全部环节按"你会在哪一刻需要它"排一遍：一个新项目要先定的是渲染方式而不是组件；一条请求进来之后依次经过 Nitro 的服务端通路、服务端渲染、payload、水合；改不动的东西几乎都卡在三个位置之一——配置读取顺序、目录约定、构建期与运行时的隔离；上线前的检查是 `.output` 独立于 `nuxt.config` 这一事实的推论。这张地图的用法是**倒着查**：从症状定位环节，再从环节找那一节的标题。

categories:
  - 前端开发

tags:
  - nuxt4
  - 知识地图
  - 架构
  - 排查顺序
  - 上线检查
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 第一层：新项目要定的不是组件

顺序是这样，反过来的话后面会返工：

1. **渲染方式**：`ssr`、`routeRules` 里的缓存与重定向、要不要预渲染。它决定了 payload、水合、性能指标的一切后续判断。
2. **目录是否用默认**：`srcDir` 默认 `app`，`dir` 那一节可以整组改；改了要同步 IDE 调试配置里的 `webRoot`。
3. **配置从哪里来**：`nuxt.config` 是唯一入口，**`nitro.config`、`vite.config`、`postcss.config`、`webpack.config` 这些独立文件已经不再被读取**，各写进对应配置节。
4. **扩展从哪里来**：模块数组、项目里的 `modules/` 目录、图层 `extends`，三者的边界分别是"跑代码"、"仓库内自动注册"、"继承文件与配置"。
5. **运行时配置怎么给**：`runtimeConfig` 与 `app.config` 是两条不同的路，前者区分 public 与服务端私有、能用环境变量覆盖，后者是应用级、可被客户端读。

## 第二层：一条请求走过的路

```txt
浏览器请求
  → Nitro 服务端通路（request → 中间件 → 服务端路由或 SSR）
  → 拼出 HTML（head、样式、入口脚本、payload）
  → 浏览器接手（水合：把事件挂到已经存在的 DOM 上）
  → 客户端导航（此后走路由、组件与数据获取的客户端分支）
```

每一段都有对应的观测点：服务端那半用 `nuxt dev --inspect` 或 IDE 的 `server: nuxt` 配置（Nitro 工作线程在 `9229`，CLI 进程在 `9230`）；水合这一段的错误通常是"服务端与客户端渲染不一致"；客户端导航这一段有 chunk 加载失败这条独立通路（`emitRouteChunkError` 的四种处置）。

流式渲染改变了这条链的形状：**外壳先冲出去，之后状态与响应头就定下来了**，所以要改响应头的动作必须挪到插件里。

## 第三层：改不动的东西卡在三个位置

**一、配置读取顺序。**图层与项目配置用 defu 合并，同名按优先级覆盖：项目文件压过 `~~/layers` 自动扫描的图层，后者压过 `extends` 里的项。改配置没反应，先确认被谁覆盖，而不是怀疑有没有读。

**二、目录约定。**组件、页面、组合式函数、服务端路由都靠目录扫描注册，`components:dirs` 又是在 `app:resolve` 之内被消费的，所以更晚改目录不生效。服务端侧只有 `api`、`routes`、`middleware`、`utils` 四个子目录会被扫描。

**三、构建期与运行时的隔离。**这两侧不共享状态、代码或上下文，只有运行时配置例外。模块选项在运行时读不到是设计如此：要用 `runtimeConfig` 显式暴露；反过来 Kit 的函数不能在组件、页面、插件或 server routes 里 import。

## 第四层：性能与体积的判断入口

顺序也是三档：

1. **先分清是网络还是 DOM**。网络侧看 chunk 数量与体积、图片格式、第三方脚本；DOM 侧看水合范围、组件是否必要地参与水合。
2. **拆包手段**：每页 dynamic import（默认开）、`Lazy` 前缀、`isLazy`、组件岛屿、`hydrate-never`（配合 `stripNeverHydratedData` 把数据留在 payload 外）。
3. **构建缓存**：`experimental.buildCache` 默认关，开了以后九个配置与锁文件一动就全量重建、Nitro 永远重建、只留 10 份 tarball。

指标归属那三句可以直接当检查表用：LCP 常与图片相关、CLS 与字体 fallback 相关、INP 与第三方脚本相关；`navigationRepaint` 这一项改善的正是导航时的 INP。

## 第五层：错误、日志与可见性

| 症状 | 位置 | 处置入口 |
| --- | --- | --- |
| 接口返回错的状态码或文本 | 服务端路由 | `createError` 带状态码；`setResponseStatus`（**流式路由上会被丢弃**） |
| 页面渲染时抛错 | 服务端渲染／应用上下文 | `app:error`、`app:error:cleared` |
| chunk 加载失败 | 客户端 | `emitRouteChunkError` 四态 |
| 404 出现得太晚、应用被整个加载 | 服务端匹配 | `experimental.early404`（四种情况自动停用） |
| 日志看不见 | 开发期 | `features.devLogs` 与 `dev:ssr-logs`；TUI 里按 `e` 跳到最后一次错误 |
| 不知道是谁改了配置 | 初始化阶段 | `experimental.debugModuleMutation` |

## 第六层：上线前那几步

- **构建与预览要成对做**：`nuxt build` 之后 `nuxt preview`，流式与样式的视觉问题只能在构建产物上验，dev 下 Vite 的样式注入方式会误导你。
- **`.output` 是一个独立产物**，与 `nuxt.config` 与模块无关；改了配置就必须重建，线上行为没变多半是没重建。
- **缓存与清理**：`npx nuxt cleanup` 删构建目录、`.output`、`dist`、`node_modules/.vite`、`node_modules/.cache` 五处；**它不删依赖也不删锁文件**，那两类问题要另走包管理器。
- **每夜频道不是给生产用的**：官方对它的表述是质量与稳定版相同，但你得常回仓库看，因为有回归未被发现的小概率；`externalVue` 那一类被单独标注"不久会移除"的东西不要写进关键路径。
- **版本线**：每个主版本至少支持到下一个主版本发布后六个月；`experimental` 里的改动不受"不含破坏性变更"这条承诺约束。

## 这张地图怎么倒着用

从症状出发，三步定位：

1. 判断落在哪一侧：浏览器、服务端运行时、还是构建期进程。
2. 在上面的层里找对应那一层，读它的边界描述（隔离、覆盖顺序、目录扫描范围这三条最常被违反）。
3. 需要查细节时用官方给的三条路：`npx nuxt docs "<关键词>"`（**版本跟着你项目走**）、MCP（工具化，按页取内容）、`/llms.txt`（零配置，当索引给模型）。

## 一个收尾的判断

学框架最容易浪费时间的地方，是把力气花在"记住有多少个 API"上。这份内容想留下的其实是四条判断：**哪些事是自动的（目录扫描、自动导入、路由生成）、哪些事是配置驱动的（渲染方式、运行时配置、构建器选择）、哪些事需要写代码（模块、插件、钩子）、哪些事只能在构建期做**。这四条能覆盖日常绝大多数"这个该在哪改"的问题。

接下来的六篇离开框架本身，讲社区侧的六件事：去哪里问、怎么报一个能被修的 bug、怎么给文档和框架贡献、版本路线图怎么读、更新日志追到哪里。

## 下一篇

下一篇 720：遇到问题去哪里问：两条社区入口、四条提问建议与一个专业渠道

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/internals
- https://nuxt.com/docs/4.x/guide/going-further/builders
- https://nuxt.com/docs/4.x/api/nuxt-config
- https://nuxt.com/docs/4.x/api/commands/docs
