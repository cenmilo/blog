---
url: /posts/4a0e595386902328f890c7eefc80ee2e/
title: 079 - 目录基准全表：rootDir、srcDir、dir、typesDir 各自相对谁
date: 2026-11-07T10:00:00+08:00
updated: 2026-11-07T10:00:00+08:00
author: cmdragon

summary:
  摘要：这一系列里出现的目录类配置项有六个，官方给它们的默认值与基准各不相同，混用就是"改了没生效"的常见来源。rootDir 定义应用的根目录，默认是 /<rootDir>，官方明确它可以被覆盖（运行 nuxt ./my-app/ 会把 rootDir 设为从当前工作目录算起那个路径的绝对路径）并写了"通常不需要配置这个选项"。srcDir 定义源目录，相对路径以 rootDir 为基准，默认在 Nuxt 4 是 "app"、在 compatibilityMode 为 3 的 Nuxt 3 是 "."。serverDir 默认 /<rootDir>/server 且相对路径同样以 rootDir 为基准——注意它的基准是 rootDir 而不是 srcDir，所以服务端目录与 app/ 是并列关系。buildDir 默认 /<rootDir>/.nuxt，官方说明许多工具假定以 . 开头的是隐藏目录、如有问题可用该选项避开。typesDir 定义生成的类型与 tsconfig 文件放在哪里，官方写明它默认取你的 buildDir、与它分开是为了让项目 tsconfig.json 引用的那些配置在 Nuxt 到别处构建时仍留在原地，相对路径以 rootDir 为基准，默认 /<rootDir>/.nuxt。workspaceDir 定义工作区目录，官方说它常用于 monorepo、Nuxt 会尝试自动检测但可在此覆盖、通常不需要配置。dir 一节里那九个子项默认值多数带着 app/ 前缀（例如 dir.pages 默认 "app/pages"），而官方给这一节的建议就是"除非必要最好保持默认"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 目录配置
  - rootdir
  - srcdir
  - monorepo

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


051 讲的是"别名指向哪"。这一篇讲**这些目录本身怎么定**：六个配置项、各自的默认值、各自的**基准**。

**为什么值得单独一篇？** 因为它们的基准不是同一个——**"改了 srcDir 但 `.nuxtignore` 不生效"这类问题，全在基准这一层。**

## 六项一览：先看默认值

| 配置项 | 官方默认值 | 官方对基准的原话 |
| --- | --- | --- |
| `rootDir` | `"/<rootDir>"` | 可被覆盖：`nuxt ./my-app/` 会把它设为从当前工作目录算起的绝对路径；"**通常不需要配置这个选项**" |
| `srcDir` | **`"app"`（Nuxt 4）** / `"."`（Nuxt 3 配 `compatibilityMode: 3`） | "**如果指定的是相对路径，它相对 `rootDir`**" |
| `serverDir` | `"/<rootDir>/server"` | "**如果指定的是相对路径，它相对于你的 `rootDir`**" |
| `buildDir` | `"/<rootDir>/.nuxt"` | （无相对路径说明，默认已是绝对形态） |
| `typesDir` | `"/<rootDir>/.nuxt"` | "**如果指定的是相对路径，它会相对于你的 `rootDir`**" |
| `workspaceDir` | `"/<workspaceDir>"` | "**Nuxt 会尝试自动检测你的工作区目录**，不过你可以在此覆盖它" |

**四个"相对路径以 `rootDir` 为基准"：`srcDir`、`serverDir`、`typesDir`，加上 071 讲 `modulesDir` 时也有一句"配置路径相对于 `options.rootDir`"。** 这一族是一致的。

## 第一条要记住的关系：`serverDir` 不在 `srcDir` 里面

050 那张地图上，`server/` 与 `app/` 是并列的。现在有了配置层面的证据：

- `srcDir` 默认 `app` → 应用在 `<rootDir>/app`
- `serverDir` 默认 `/<rootDir>/server` → **服务端在 `<rootDir>/server`**，**它的基准是 `rootDir`**

**这条关系直接决定了 078 那个写法：`.nuxtignore` 里要写 `app/pages/bar.vue`——因为那个文件的真实路径是 `<rootDir>/app/pages/bar.vue`。**

**还有一个反直觉点：** 084 会看到 `watch` 的基准是 `srcDir`（**与这四个不同**）。**同一个"相对路径"词，在不同配置项上基准不一样。**

## 第二条：`dir` 那九项为什么自带 `app/`

051 抄过 `dir` 一节的九个默认值。**现在可以把"为什么"说清了**：

| 子项 | 默认值 |
| --- | --- |
| `dir.app` | `"app"` |
| `dir.assets` | `"app/assets"` |
| `dir.layouts` | `"app/layouts"` |
| `dir.middleware` | `"app/middleware"` |
| `dir.modules` | `"modules"` |
| `dir.pages` | `"app/pages"` |
| `dir.plugins` | `"app/plugins"` |
| `dir.public` | `"public"` |
| `dir.shared` | `"shared"` |

**它们都是相对项目根的路径，所以带 `app/` 的那些就是在复述"源目录在 `app/` 下"这件事**；而 `modules`、`public`、`shared` 三项没有 `app/` 前缀——**因为它们本来就在根目录下（050 那张地图把这三个都画在与 `app/` 并列的位置）。**

**这两组并排看，`dir` 其实就是"把这六个目录配置项摊平成一份可改的表"。** 官方给这一节的建议也是同一个方向：

> 自定义 Nuxt 使用的默认目录结构。**除非必要，最好保持默认。**

## 第三条：`typesDir` 为什么单独存在

这一项的官方说明值得完整读，它解释了一个非常具体的工程场景：

> 定义生成的类型与 `tsconfig.json` 文件将被放置的目录。
>
> **这默认取你的 `buildDir`，并与它分开；分开的好处是：当 Nuxt 到别处构建时（例如构建一个已经在开发模式下运行过的项目），你项目 `tsconfig.json` 所引用的那些配置仍然留在原地。**
>
> 如果指定的是相对路径，它会相对于你的 `rootDir`。

**翻译成一句话：`buildDir` 可以搬去别处构建，而类型引用不能跟着搬走**——所以两者虽然默认同一个位置，却是两个独立可配的键。

**这与 076 那份根 `tsconfig.json` 正好咬合**：它里面写的四条 `references` 指向 `./.nuxt/tsconfig.app.json` 这种路径——**如果生成的类型目录跟着构建目录跑了，那四条引用就断了。`typesDir` 的存在就是防止这件事。**

## 第四条：`workspaceDir` 与 monorepo

> 定义你应用的工作区目录。
>
> **在 monorepo 设置里通常会用到它。Nuxt 会尝试自动检测你的工作区目录，不过你可以在此覆盖它。通常不需要配置这个选项。**

**这条与 071 的 `modulesDir` 是一对**：官方在那边写的是"**如果你的项目组织成 yarn workspace 风格的 monorepo，设置这个字段可能是必要的**"。

**于是 monorepo 里有两个"自动检测可能不够、需要手动指"的键**：`workspaceDir`（工作区在哪）与 `modulesDir`（拿哪些目录做路径解析、类型是数组可给多个）。**而 068 的图层、067 的 `extends`、021 的 `--cwd` 是另外三条与 monorepo 相关的通道。**

## 把 051 那张别名表叠回来：两套机制、一件事

`alias` 表里有两行值得与这里对照：

```json
{
  "~": "/<rootDir>/app",
  "~~": "/<rootDir>"
}
```

**注意它是硬编码到 `<rootDir>/app` 的绝对形态。** 于是有了一件容易踩的事：

| 你在改 | 影响的是 | 是否自动带动 |
| --- | --- | --- |
| `srcDir` | 源码实际位置、`.nuxtignore` 前缀（078）、`add-template` 的生成路径（046）、`app.config.ts` 该放哪（055） | —— |
| `alias['~']` | `~/...` 这个写法解析到哪 | **它默认写死为 `<rootDir>/app`** |

**两者都写着 `app`，但它们是两份配置**（官方在两处各自给了默认值，这一节没有说明它们的联动关系）。**所以真要动目录结构时，两处都要核对**——这也是我把官方"除非必要，最好保持默认"那句重复一遍的原因：**改一处而漏另一处，症状就是"文件明明在那儿，`~/` 却指到别处"。**（后一句症状是我按两张默认值表推的判断，不是官方写在这里的话。）

## 自查

- [ ] 六个目录配置项里，哪几个的相对路径基准是 `rootDir`？哪一个不是？
- [ ] 为什么 `.nuxtignore` 里的模式要带 `app/`？用两个配置项的名字解释它。
- [ ] `dir.pages` 的默认值是什么？`dir.shared` 呢？为什么一个带 `app/` 一个不带？
- [ ] `typesDir` 与 `buildDir` 默认相同，为什么要分成两个键？
- [ ] monorepo 场景下，官方点名"可能需要设置"的是哪两个键？
- [ ] 如果你只改了 `srcDir`，哪些篇目讲过的东西会连带受影响？

## 下一篇

080 讲这份配置文件的边界：`rootDir` 能被命令行位置参数覆盖、`buildDir` 换名会牵动 `.gitignore` 与 `nuxt.lock` 的位置，而 077 说的那份 `.gitignore` 清单里为什么恰好有 `.nuxt`。

## 本文依据

- `api/nuxt-config`（rootDir：定义应用的根目录、可被覆盖例如运行 nuxt ./my-app/ 会把 rootDir 设为从当前/工作目录算起该路径的绝对路径、通常不需要配置这个选项、类型 string、默认 "/<rootDir>"。srcDir：定义 Nuxt 应用的源目录、指定相对路径时相对 rootDir、类型 string、默认 "app"（Nuxt 4）与 "."（Nuxt 3 配 compatibilityMode: 3）、示例 srcDir: 'app/' 及其期望目录结构。serverDir：定义服务端目录存放 Nitro 路由/中间件/插件、相对路径相对于 rootDir、类型 string、默认 "/<rootDir>/server"。buildDir：定义已构建的 Nuxt 文件放置目录、许多工具假定以 . 开头的是隐藏目录如有问题可用此选项避免、类型 string、默认 "/<rootDir>/.nuxt"、示例 buildDir: 'nuxt-build'。typesDir：定义生成的类型与 tsconfig.json 文件放置目录、默认取 buildDir 并与之分开的理由（当 Nuxt 到别处构建如构建一个已在开发模式下运行过的项目时，项目 tsconfig.json 引用的配置仍留在原地）、相对路径相对 rootDir、类型 string、默认 "/<rootDir>/.nuxt"。workspaceDir：定义应用的工作区目录、monorepo 常用、Nuxt 会尝试自动检测但可在此覆盖、通常不需要配置、类型 string、默认 "/<workspaceDir>"。dir：自定义 Nuxt 使用的默认目录结构且"除非必要最好保持默认"，九个子项默认值 app=app、assets=app/assets（构建中别名为 ~assets）、layouts=app/layouts、middleware=app/middleware、modules=modules、pages=app/pages、plugins=app/plugins、public=public（静态文件可通过 Nuxt 服务器直接访问并在应用生成时复制进 dist 文件夹）、shared=shared（在应用与服务端之间共享）。alias 默认值含 "~": "/<rootDir>/app" 与 "~~": "/<rootDir>"。modulesDir：用于设置做路径解析时使用的 modules 目录、配置路径相对于 options.rootDir（默认为当前工作目录）、yarn workspace 风格的 monorepo 可能需要设置、类型 array、默认 ["<rootDir>/node_modules"]，见 071）
- `directory-structure`（对照用：根目录是包含 nuxt.config.ts 的目录；app/ 为主目录、public/、server/、shared/、modules/、layers/ 与 app/ 并列出现在同一份地图上，见 050）
- `directory-structure/nuxtignore`（对照用：基准是 rootDir、示例模式全部以 app/ 开头，见 078）
- `directory-structure/tsconfig`（对照用：项目根的 tsconfig.json 只含 files: [] 与四条指向 .nuxt/*.json 的 references，见 076）
- `api/commands/add-template` 与 `directory-structure/app/app-config`（对照用：生成的文件相对 srcDir 写出、srcDir 默认为 app/；配置了自定义 srcDir 时要把 app.config 放在新 srcDir 的根部，见 046、055）
