---
url: /posts/963c160137cda3cc34a37c7da472a6fd/
title: 050 - 目录树全景：Nuxt 项目里每个位置各自决定什么
date: 2026-10-28T11:00:00+08:00
updated: 2026-10-28T11:00:00+08:00
author: cmdragon

summary:
  摘要：官方文档的目录结构那一节给每个目录都留了一句定义，把它们连起来读就是整个框架的骨架：根目录是"包含 nuxt.config.ts 的那个目录"；app/ 是应用的主目录，含 assets、components、composables、layouts、middleware、pages、plugins、utils 八个小目录与 app.config.ts、app.vue、error.vue 三个特定文件；public/ 里的文件在根路径提供且不被构建过程修改，适合必须保住文件名的东西（官方举 robots.txt）或大概率不会变的（favicon.ico）；server/ 放服务端代码，含 api、routes、middleware、plugins、utils；shared/ 是唯一同时被 Vue 应用与 Nitro 服务器使用的目录；test/ 是官方推荐的测试位置；content/ 由 Nuxt Content 模块启用；modules/ 放本地模块；layers/ 里的图层会被自动注册。这一节还有一句常被忽略的话——它讲的整套结构里并没有 .nuxt 与 .output，那两棵是生成物，不是源码结构。

categories:
  - 前端开发

tags:
  - nuxt4
  - 目录结构
  - app
  - server
  - shared

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前面 49 篇把命令走完了。从这一篇起换一个问题：**代码该放在哪儿。**

005 讲过"约定的代价"，017 讲过"哪些魔法会咬人"。这一篇把那套约定摊成一张完整的地图——**每一项都用官方在目录结构那一节给出的原话。**

## 官方怎么给这件事定性

> Nuxt 应用有一套特定的目录结构，用来组织代码。**这套结构被设计为易于理解，并以一致的方式使用。**

这一节没有给你命令，它给的是**十一个位置的一句话定义**。而"每个目录各自决定什么"这件事，正是后面 054-069 这一整段的目录。

## 根目录：`nuxt.config.ts` 在哪儿，哪儿就是根

官方的第一条定义非常具体：

> Nuxt 应用的根目录，是**包含 `nuxt.config.ts` 文件的那个目录**。这个文件用来配置 Nuxt 应用。

**注意这个定义方式的含义：根目录不是"项目文件夹"这个直觉，而是"配置文件的所在地"。** 051 会看到 `rootDir` 这个配置项，而 048 讲过 `nuxt ./my-app/` 会把 `rootDir` 指过去——**这三句说的是同一件事的不同侧面。**

## `app/`：八个目录 + 三个文件

官方称它是"应用的**主目录**"，然后逐条给了定义。我把这八句原话整理成表（右侧完全照抄官方描述）：

| 目录 | 官方的定义 |
| --- | --- |
| `assets/` | 构建工具（Vite 或 webpack）会处理的网站资源 |
| `components/` | 应用的 Vue 组件 |
| `composables/` | 添加你的 Vue 组合式函数 |
| `layouts/` | 包装在你页面之外、**避免在页面之间重新渲染**的 Vue 组件 |
| `middleware/` | 在导航到某个路由之前运行代码 |
| `pages/` | 为你的 Web 应用创建路由的**基于文件的路由** |
| `plugins/` | 在**你的 Nuxt 应用创建之时**使用 Vue 插件及更多内容 |
| `utils/` | 添加遍布你的应用的函数，可在组件、组合式函数与页面里使用 |

这份清单里有两个定义值得停下来看：

**一、`layouts/` 的定义里带着一个性能承诺。**"避免在页面之间重新渲染"——026 讲布局时我只说了它是"页面的包装器"，官方在这里给的理由还包括**它不会因为换页而重渲染**。

**二、`plugins/` 的定义把时机写进了名字里。**"在应用**创建之时**"——这正是 019 生命周期里那个位置（插件在应用实例创建阶段运行）。

然后官方补了这一层里的三个**特定文件**：

- `app.config.ts`：你应用内部的一个**响应式**配置
- `app.vue`：你 Nuxt 应用的**根组件**（025 已经把它当入口点讲过）
- `error.vue`：你 Nuxt 应用的**错误页**

## `public/`：不被构建修改的那一批

> `public/` 目录包含 Nuxt 应用的公共文件。**这里的文件在根路径提供，且不被构建过程修改。**
>
> 这适合那些**必须保住自己名字**的文件（例如 `robots.txt`）*或者*大概率不会变的文件（例如 `favicon.ico`）。

**"不被构建过程修改"这半句是分界线。** 064 会专门讲 `assets/` 与 `public/` 的差别（一处会被重写路径、一处不会），而官方在这里给的判据已经够用：**名字要稳定 → `public/`；要参与打包与哈希 → `assets/`。**

## `server/`：五个子目录，服务端那一侧

> `server/` 目录包含 Nuxt 应用的服务端代码。

它的五个子目录（官方定义照抄）：

| 子目录 | 官方的定义 |
| --- | --- |
| `api/` | 包含应用的 **API 路由** |
| `routes/` | 包含应用的服务端路由（例如动态的 `/sitemap.xml`） |
| `middleware/` | 在一个服务端路由被处理之前运行代码 |
| `plugins/` | 在**你的 Nuxt 服务器创建之时**使用插件及更多内容 |
| `utils/` | 添加遍布你的应用的函数，可用于你的服务端代码 |

**这里有一个和 `app/` 层同名但完全不同的东西：`middleware` 与 `plugins`。**

- `app/middleware/` = 导航到某个**路由**之前（客户端路由那一侧，033 的追踪时间线里那条叫 route middleware）
- `server/middleware/` = 一个**服务端路由被处理**之前

同理 `plugins`：`app/plugins/` 在**应用创建时**、`server/plugins/` 在**服务器创建时**。**060 与 065 会分别展开，但你从这两个定义的差别就能看出它们不是同一层东西。**

（027 那个 `render:html` 的插件文件放在 `server/plugins/`，正是这张表里的第四行。）

## `shared/`：唯一被两侧共用的位置

> `shared/` 目录包含 Nuxt 应用**与 Nuxt 服务器共享**的代码。**这份代码可以在 Vue 应用与 Nitro 服务器里同时使用。**

**这句"两侧同时可用"是 `shared/` 存在的理由，也是它唯一的特殊性**——其它目录都只归一侧（`server/` 归服务端、`app/` 归应用）。066 会展开它的自动导入边界。

## 剩下四个位置

| 位置 | 官方的定义 |
| --- | --- |
| `test/` | 应用测试（**单元、Nuxt 运行时与端到端**）的**推荐**位置 |
| `content/` | 由 **Nuxt Content 模块启用**；用于以 Markdown 文件为应用搭建基于文件的 CMS |
| `modules/` | 包含 Nuxt 应用的**本地模块**；模块用来扩展应用的功能 |
| `layers/` | 让你组织并共享可复用的代码、组件、组合式函数与配置；**这个目录里的图层会被自动注册进你的项目** |

这一张表里有两句要紧的话：

- **`test/` 是"推荐"而不是"要求"**——官方用的词是 recommended。而"哪些测试目录会被纳入类型上下文"另有规则（069）。
- **`content/` 不是框架自带的**，它由模块启用。**换句话说：你项目里没有 `content/` 是完全正常的**（对照 045：模块要经过 `nuxt add` 才进来）。
- **`layers/` 这一句"自动注册"值得留意**——045 讲过图层会被写进 `nuxt.config` 的 `extends`；而那句说的是**这个目录下的图层自动注册**，两条路并存（它们怎么合起来生效，068 讲）。

## 三个配置文件

官方最后把三样文件单列出来：

- **`nuxt.config.ts`**：Nuxt 应用的**主配置文件**
- **`.nuxtrc`**：**另一种配置 Nuxt 应用的语法**（官方注明"对全局配置有用"）
- **`.nuxtignore`**：**在构建阶段忽略根目录里的文件**

这三条定义合起来是一条容易被漏掉的事实：**配置不只有一份文件，而且三份的作用域不同**（`nuxt.config` 在项目里、`.nuxtrc` 适合放全局、`.nuxtignore` 管构建阶段的忽略）。018 讲第一份，074 与 078 讲后两份。

## 那份清单没列什么：生成物不在目录结构里

这是本篇我最想让你注意到的一点：**`app/`、`public/`、`server/`、`shared/`、`test/`、`content/`、`modules/`、`layers/` 里，没有 `.nuxt`，也没有 `.output`。**

**这不是遗漏。** 它讲的是**你写的代码放在哪儿**；`.nuxt` 与 `.output` 是**框架的产物**（035/052 讲前者、053 讲后者）。区分这两类位置有个直接好处：

- **生成物可以删**（`nuxt cleanup` 删得掉、`nuxt dev` 会重建，040 与 052）；
- **源码结构不能随便挪**——挪了框架就不认（这份清单定义的就是"认不认"）。

## 自查

- [ ] 官方的"根目录"定义是按什么判定的？
- [ ] `layouts/` 的定义里除了"包装页面"还带了什么承诺？
- [ ] `app/middleware/` 与 `server/middleware/` 的时机差别是什么？`plugins` 同理吗？
- [ ] `public/` 的判据是哪一句？`robots.txt` 该放哪、为什么？
- [ ] 哪个目录是官方明确说"两侧共用"的？哪个目录不是框架自带的？
- [ ] `.nuxt` 与 `.output` 为什么不出现在这份清单里？

## 下一篇

051 讲这些位置怎么被"改名"与"引用"：`alias` 的十项默认值（`~` 到底指到哪）、`srcDir` 为什么默认是 `app`、以及 `dir` 那九个子项各自能改成什么。

## 本文依据

- `directory-structure`（开篇"这套结构被设计为易于理解并以一致方式使用"；Root Directory：包含 nuxt.config.ts 的目录、该文件用于配置应用；App Directory 为应用主目录并列出 assets（构建工具 Vite 或 webpack 会处理的网站资源）、components（应用的 Vue 组件）、composables（添加你的 Vue 组合式函数）、layouts（包装在页面之外且避免在页面之间重新渲染的 Vue 组件）、middleware（在导航到某个路由之前运行代码）、pages（基于文件的路由为 Web 应用创建路由）、plugins（在 Nuxt 应用创建之时使用 Vue 插件及更多内容）、utils（遍布应用的函数可在组件/组合式函数/页面中使用），以及三个特定文件 app.config.ts（应用内的响应式配置）、app.vue（Nuxt 应用的根组件）、error.vue（错误页）；Public Directory：包含公共文件、文件在根路径提供且不被构建过程修改、适合必须保住名字的文件（如 robots.txt）或大概率不会变的（如 favicon.ico）；Server Directory：包含服务端代码并列出 api（应用的 API 路由）、routes（服务端路由，例如动态的 /sitemap.xml）、middleware（在服务端路由被处理之前运行代码）、plugins（在 Nuxt 服务器创建之时使用插件及更多内容）、utils（可用于服务端代码的函数）；Shared Directory：包含应用与 Nuxt 服务器共享的代码，可在 Vue 应用与 Nitro 服务器同时使用；Test Directory：应用测试（单元、Nuxt 运行时与端到端）的推荐位置；Content Directory：由 Nuxt Content 模块启用，用于以 Markdown 文件搭建基于文件的 CMS；Modules Directory：包含本地模块，模块用于扩展应用功能；Layers Directory：组织并共享可复用代码/组件/组合式函数/配置，目录内的图层会被自动注册进项目；Nuxt Files 三条：nuxt.config.ts 为主配置文件、.nuxtrc 是另一种配置语法（对全局配置有用）、.nuxtignore 用于在构建阶段忽略根目录中的文件）
