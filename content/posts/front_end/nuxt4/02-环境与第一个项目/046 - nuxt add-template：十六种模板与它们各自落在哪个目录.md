---
url: /posts/e9cdd91827b673734b60063b10a33caf/
title: nuxt add-template：十六种模板与它们各自落在哪个目录
date: 2026-10-27T10:00:00+08:00
updated: 2026-10-27T10:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt add-template 的官方定义是"把一个文件脚手架进你项目结构所对应的位置"，它取代 nuxt add <template> <name>（旧写法仍能跑但已废弃）。它的第一个参数 TEMPLATE 一共十六个取值：api、app、app-config、component、composable、error、layer、layout、middleware、module、page、plugin、server-middleware、server-plugin、server-route、server-util；第二个参数 NAME 是生成文件的名字。文档给了九个小节与生成路径实例，例如 component TheHeader 生成 app/components/TheHeader.vue、page "category/[id]" 生成 app/pages/category/[id].vue、api hello 生成 server/api/hello.ts、layer subscribe 生成 layers/subscribe/nuxt.config.ts。规则里有两条最容易忽略：一是所有路径都相对 srcDir（默认为 app/）写、服务端文件相对 serverDir 写；二是 --mode 与 --method 的每个取值同时也是一个独立开关，所以 --mode client 与 --client 是同一件事，而 component 与 plugin 才吃 --mode、api 才吃 --method。最后官方加了一条安全限制：会解析到项目之外的名字会被拒绝，因此要传不带前导斜杠与 .. 段的相对路径。

categories:
  - 前端开发

tags:
  - nuxt4
  - 脚手架
  - 目录结构
  - 命令行
  - 模块

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


045 末尾那条分界落地在这一篇：**新建一个文件，用 `nuxt add-template`。**

这一条命令的价值不在"少打几行模板代码"，而在于**它替你决定路径**——050 会看到 Nuxt 的目录是靠位置生效的，放错目录等于没写。

## 官方定义与那条废弃说明

> `add-template` 命令把一个实体脚手架进**你项目结构所对应的位置**。它取代 `nuxt add <template> <name>`，**后者仍然能跑，但已废弃。**

一句提醒：**"仍能跑"是它兼容旧习惯，不是它等于新写法。** 新代码一律写 `add-template`。

## 用法行

```bash [Terminal]
npx nuxt add-template <TEMPLATE> <NAME> [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--force] [--mode=<client|server>] [--method=<connect|delete|get|head|options|patch|post|put|trace>] [--global] [--api] [--pages] [--client] [--server] [--connect] [--delete] [--get] [--head] [--options] [--post] [--put] [--trace] [--patch]
```

两个位置参数：

| 参数 | 官方说明 |
| --- | --- |
| `TEMPLATE=<api\|app\|app-config\|component\|composable\|error\|layer\|layout\|middleware\|module\|page\|plugin\|server-middleware\|server-plugin\|server-route\|server-util>` | 指定生成哪种模板 |
| `NAME` | 指定生成文件的名字 |

**这十六个取值就是这一条命令的全部可生成对象。**

## 路径规则：一切都相对 `srcDir`

官方在举例之前，先写了这条全局规则：

> 文件相对于你的 [`srcDir`](https://nuxt.com/docs/4.x/api/nuxt-config#srcdir) 写出，`srcDir` 默认为 `app/`；**服务端文件相对于你的 [`serverDir`](https://nuxt.com/docs/4.x/api/nuxt-config#serverdir) 写出。下面这些路径都假定它们处于默认值。**

这一句决定了下面所有例子的可读性——**如果你的 `srcDir` 改过，那"生成到 `app/xxx`"这些路径要整体换算**（`srcDir` 本身在 051 讲）。

## 十六种模板里，文档给了九个小节

我把官方带示例的九个整理成表（右侧的生成路径全是文档原话）：

| 模板 | 命令 | 官方给的生成结果 | 可加的修饰符 |
| --- | --- | --- | --- |
| `component` | `nuxt add-template component TheHeader` | `app/components/TheHeader.vue` | `--mode`，或 `--client`/`--server` |
| `composable` | `nuxt add-template composable foo` | `app/composables/foo.ts` | （无列出） |
| `layout` | `nuxt add-template layout custom` | `app/layouts/custom.vue` | （无列出） |
| `plugin` | `nuxt add-template plugin analytics` | `app/plugins/analytics.ts` | `--mode`，或 `--client`/`--server` |
| `page` | `nuxt add-template page about` | `app/pages/about.vue` | （无列出） |
| `middleware` | `nuxt add-template middleware auth` | `app/middleware/auth.ts` | `--global` |
| `api` | `nuxt add-template api hello` | `server/api/hello.ts` | `--method`，或把方法名直接当开关（`--get`、`--post` 等） |
| `server-route` | `nuxt add-template server-route webhook` | `server/routes/webhook.ts` | `--api`：把路由写到 `server/api` 下而不是 `server/routes` |
| `layer` | `nuxt add-template layer subscribe` | `layers/subscribe/nuxt.config.ts` | （无列出） |

**另外七个取值（`app`、`app-config`、`error`、`module`、`server-middleware`、`server-plugin`、`server-util`）在这一页没有独立小节和示例。** 我不替它们猜生成路径——想要精确结果，跑一次看它把文件放哪，或者用下面的安全规则去推断它落在项目内的相对位置。

两个值得单独看的例子。

**一、动态段要用引号包。**

```bash [Terminal]
# Generates `app/pages/category/[id].vue`
npx nuxt add-template page "category/[id]"
```

**路径里带 `[id]` 这种方括号时，整个值要作为 `NAME` 传进去并加引号**——057、058 会讲这些方括号段怎么变成路由参数。

**二、修饰符能生成带后缀的文件名。**

```bash [Terminal]
# Generates `app/plugins/sockets.client.ts`
nuxt add-template plugin sockets --mode client
```

## 修饰符的对称规则（最容易记错的一条）

官方的"Modifiers"整段：

> 有些模板接受一个额外的开关，它会给生成的文件名加上后缀（比如 `.client` 或 `.get`）。**`--mode` 与 `--method` 接受的每一个取值，本身也各是一个开关，所以 `--mode client` 与 `--client` 做的是同一件事。**

这一条有两个用途：

1. **写法二选一，结果相同。** 表格里的九个里只有 `component`/`plugin`（`--mode`）与 `api`（`--method`）带修饰符——**给 `composable` 传 `--client` 是没有意义的**（官方只在这些模板下标了 Modifier flags）。
2. **`--mode` 的作用对象是明确的**：官方选项表写着"给**组件或插件**加上 client 或 server 后缀"。**不是所有模板都能加。**

`--method` 能吃九种 HTTP 方法（`connect|delete|get|head|options|patch|post|put|trace`），每一种同时是独立开关（`--get`、`--post`……）。**它只给 API 路由用。**

`--server-route` 的 `--api` 是个反方向的修饰符：默认写到 `server/routes`，加上 `--api` 就写到 `server/api` 下面。

## 剩下几个开关

| 选项 | 默认 | 官方说明 |
| --- | --- | --- |
| `--cwd=<directory>` | `.` | 指定项目根目录 |
| `--logLevel` | | 构建期日志级别 |
| `--force` | `false` | **文件已存在时覆盖它** |
| `--global` | | 创建全局路由中间件 |
| `--api` | | 在服务端路由里创建到 API 目录下 |
| `--pages` | | 在 app 模板里包含 `NuxtPage` 与 `NuxtLayout` |

三条各说一句：

- **`--force` 默认是 `false`**，也就是**默认不覆盖已有文件**。这是这条会往目录里写东西的命令里最重要的保护，别习惯性加它。
- **`--global` 对应 `middleware`**：生成的仍然是 `app/middleware/auth.ts` 这类文件，但按中间件规则，全局中间件靠**文件名后缀**表达（060 会逐条讲 `.global` 那一套）。这里只知道官方把它做成一个开关。
- **`--pages` 是给 `app` 模板用的**：官方措辞"在 app 模板里包含 NuxtPage 与 NuxtLayout"。**这正好接上 025 那个坑**——`app.vue` 里没有 `<NuxtPage />` 时页面不显示；用这个开关生成的外壳就带着它。

## 一条安全限制：名字不能解析到项目外

官方在页末用 note 写了这条，读的时候别滑过去：

> **会解析到项目之外的名字会被拒绝**，所以请传入相对于项目、不带前导斜杠也不带 `..` 段的路径。

这就是为什么 `NAME` 必须是**项目内的相对路径**。**反例形态就是它明确拒绝的那两种：前导斜杠（`/etc/xxx`）、以及 `..` 段（`../../xxx`）。** 换句话说，你不能靠这个参数把文件写到项目外面去。

## 自查

- [ ] `TEMPLATE` 一共几个取值？`server-util` 在里面吗？
- [ ] 文档给的九个示例里，`api` 生成到哪、`server-route` 默认生成到哪、加 `--api` 后呢？
- [ ] `--mode client` 与 `--client` 的等价性，官方是怎么表述的？哪些模板能吃 `--mode`？
- [ ] 你要生成 `app/pages/category/[id].vue`，命令怎么写？为什么要加引号？
- [ ] 目标文件已存在时默认发生什么？`NAME` 能写绝对路径吗？

## 下一篇

047 讲两条"查资料与开面板"的命令：`nuxt docs` 搜的是**你项目所用版本**的文档而不是线上最新版，`nuxt devtools enable` 则会把偏好写进用户级的 `.nuxtrc`。

## 本文依据

- `api/commands/add-template`（add-template 把实体脚手架进项目结构对应位置、取代 nuxt add <template> <name> 且后者仍能运行但已废弃；用法行与全部开关；参数 TEMPLATE 的十六个取值 api|app|app-config|component|composable|error|layer|layout|middleware|module|page|plugin|server-middleware|server-plugin|server-route|server-util 与 NAME；选项 --cwd 默认 .、--logLevel、--force 默认 false（Overwrite the file if it already exists）、--mode=<client|server>（给 component 或 plugin 加 client/server 后缀）、--method=<connect|delete|get|head|options|patch|post|put|trace>（给 API 路由加 HTTP 方法后缀）、--global（Create global route middleware）、--api（在服务端路由里写入 API 目录）、--pages（在 app 模板中 Include NuxtPage and NuxtLayout）、--client/--server 分别是 --mode client|server 的简写、--connect..--patch 分别是对应 --method 的简写；Modifiers 段：某些模板接受额外开关给文件名加后缀（如 .client 或 .get），--mode 与 --method 的每个取值本身也是一个开关故 --mode client 与 --client 等效；文件相对 srcDir 写出（默认 app/）、服务端文件相对 serverDir 写出、示例路径均假定默认值；示例 plugin sockets --mode client 生成 app/plugins/sockets.client.ts；九个小节示例：component TheHeader→app/components/TheHeader.vue（Modifier flags --mode 或 --client/--server）、composable foo→app/composables/foo.ts、layout custom→app/layouts/custom.vue、plugin analytics→app/plugins/analytics.ts（--mode）、page about→app/pages/about.vue 与 page "category/[id]"→app/pages/category/[id].vue、middleware auth→app/middleware/auth.ts（--global）、api hello→server/api/hello.ts（--method 或方法名开关）、server-route webhook→server/routes/webhook.ts（--api 可写到 server/api）、layer subscribe→layers/subscribe/nuxt.config.ts；note 会解析到项目之外的名字被拒绝，需传不带前导斜杠与 .. 段的项目相对路径）
- `api/nuxt-config`（srcDir 默认 app（Nuxt 4）且相对路径以 rootDir 为基准、dir.public 与 dir.shared 等默认值、serverDir 默认 <rootDir>/server——用于解释生成路径的基准）
- `api/commands/add`（note：nuxt add <template> <name> 已废弃应改用 nuxt add-template）
