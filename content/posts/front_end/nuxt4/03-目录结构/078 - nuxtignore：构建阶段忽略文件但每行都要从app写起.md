---
url: /posts/c57825fd3f8d2fade194d7d53d2db6f7/
title: .nuxtignore：构建阶段忽略文件，但每行都要从 app/ 写起
date: 2026-11-07T09:00:00+08:00
updated: 2026-11-07T09:00:00+08:00
author: cmdragon

summary:
  摘要：.nuxtignore 告诉 Nuxt 在构建阶段忽略你项目根目录（rootDir）里的文件，它与 .gitignore 和 .eslintignore 遵循同一套规范——每一行是一个 glob 模式，指明哪些文件应当被忽略；官方还给了更细的三条替代入口，可以在 nuxt.config 里配置 ignoreOptions、ignorePrefix 与 ignore。这一页值得逐字读的是那份用法示例，因为官方写的每一条模式都以 app/ 开头：app/layouts/foo.vue 忽略某个布局、app/layouts/*-ignore.vue 忽略以 -ignore.vue 结尾的布局文件、app/pages/bar.vue 忽略某个页面、app/pages/ignore/*.vue 忽略某目录下的页面、app/middleware/foo/*.js 配一行 !app/middleware/foo/bar.js 表示"忽略 foo 目录下的中间件但除外 bar.js"。结合 051 抄过的 srcDir 默认值 "app"（相对路径以 rootDir 为基准），就能明白这里的 .nuxtignore 与 .gitignore 最大的实操差别：它按 Nuxt 实际看到的目录层级来写，所以要带着 app/ 这一段。至于示例里那个 ignore 目录，它只是一个普通文件夹名，被手写规则排除掉而已。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxtignore
  - glob
  - 构建
  - 配置

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


077 末尾我把两个文件摆在一起对比过：`.gitignore` 管 git，`.nuxtignore` 管 Nuxt 的构建阶段。**这一篇讲后者，重点只有一件事——它的路径要从 `app/` 写起。**

## 定义：作用域与语法都写在两句里

> `.nuxtignore` 文件让 Nuxt 在你的**构建阶段**忽略你项目**根目录**（`rootDir`）里的文件。
>
> 它遵循与 `.gitignore` 和 `.eslintignore` 文件**相同的规范**——**每一行是一个 glob 模式**，指明应当忽略哪些文件。

**两句拆出四个要素：**

| 要素 | 官方的值 |
| --- | --- |
| 生效阶段 | **构建阶段**（不是运行时、不是提交时） |
| 基准 | **`rootDir`**（079 会看到它的默认与可覆盖方式） |
| 语法 | 与 `.gitignore`/`.eslintignore` 同一套规范 |
| 单位 | **一行一个 glob** |

**官方在页末又指了一次规范来源**（read-more 到 git 的 gitignore 文档）——**所以 `!` 除外、`*` 通配这些写法在这里同样成立**，下面示例里正好有一条。

## 那份示例：每条都带 `app/`

```bash [.nuxtignore]
# ignore layout foo.vue
app/layouts/foo.vue
# ignore layout files whose name ends with -ignore.vue
app/layouts/*-ignore.vue

# ignore page bar.vue
app/pages/bar.vue
# ignore page inside ignore folder
app/pages/ignore/*.vue

# ignore route middleware files under foo folder except foo/bar.js
app/middleware/foo/*.js
!app/middleware/foo/bar.js
```

**五行注释对应的能力，正好覆盖日常所有形态：**

| 模式 | 官方注释 | 用到的语法 |
| --- | --- | --- |
| `app/layouts/foo.vue` | 忽略某个布局 | 精确路径 |
| `app/layouts/*-ignore.vue` | 忽略名字以 `-ignore.vue` 结尾的布局文件 | 段内通配 |
| `app/pages/bar.vue` | 忽略某个页面 | 精确路径 |
| `app/pages/ignore/*.vue` | 忽略 `ignore` 文件夹里的页面 | 目录 + 一层通配 |
| `app/middleware/foo/*.js` + `!app/middleware/foo/bar.js` | 忽略 `foo` 目录下的中间件，**但除外 `foo/bar.js`** | 通配 + **感叹号例外** |

**最后这一对是这一页最实用的写法**：**先大面积忽略、再点名放行**——和 077 那三行 `!.env.example` 是同一个机制。

## 关键一条：为什么每行都以 `app/` 开头

**注意这份示例里没有一个模式是 `layouts/foo.vue` 这种"从 `src/` 里那层直接写起"的形态**——**全部以 `app/` 开头。**

原因在 051 抄过的那条：

> `srcDir`——定义你 Nuxt 应用的源目录。**如果指定的是相对路径，它相对 `rootDir`。** 默认 **`"app"`（Nuxt 4）**。

**两条合起来就是一条可执行规则**（我按上述两处官方事实得出的结论）：

- **`.nuxtignore` 的基准是 `rootDir`；**
- **而 `app/pages/bar.vue` 这个文件的实际路径是 `<rootDir>/app/pages/bar.vue`。**

**所以你要忽略它，模式就得写 `app/pages/bar.vue`。写成 `pages/bar.vue` 是不匹配的**——**"我加了 `.nuxtignore` 但那个页面照样能被访问到"就是这个原因**（这一句是我对上述两条事实的组合推论，官方没在这里明说"否则会怎样"）。

**如果你改过 `srcDir`（051、079），这些模式的前缀要跟着换。**

## 那个 `ignore` 目录：只是文件夹名

第三组的第二条注释是 "ignore page inside ignore folder"：

```
app/pages/ignore/*.vue
```

**别把它读成"名字叫 `ignore` 的目录会被自动忽略"。** 按这一页给出的规则，**它是被你亲手写下的这条模式排除掉的普通目录**——换一个名字，模式跟着换就行。

**"目录名自带特殊含义"这类直觉，在这一系列里已经吃过两次了**（017 讲约定的代价、096 讲 `<ClientOnly>` 的真实机制）。**这里官方给的只有"每行是一个 glob"这一条规则。**

## 三条替代入口：官方给的是 tip

> （tip）你也可以在你的 `nuxt.config` 文件里配置 `ignoreOptions`、`ignorePrefix` 与 `ignore`。

**这三个配置项值得先认识一下名字，因为 080 与 082 会逐个展开：**

| 配置项 | 它管什么（预告） |
| --- | --- |
| `ignore` | **一组 glob 模式**（不只有一个文件里的那些） |
| `ignorePrefix` | **一个前缀约定**：文件名带这个前缀就被忽略 |
| `ignoreOptions` | **传给底层忽略实现的选项** |

**为什么要 `ignorePrefix` 这么个东西？** 因为 `.nuxtignore` 是集中式清单，而"某个文件本身就该被排除"更适合**就地标记在文件名上**（082 会看到默认前缀是什么，以及官方限定它作用于哪些目录）。

**所以这三条与 `.nuxtignore` 是同一件事的两种风格**：**集中列清单** vs **就地打标**。

## 自查

- [ ] `.nuxtignore` 在什么阶段生效？它的基准目录是哪个？
- [ ] 它的语法与哪两个文件同规范？单位是什么？
- [ ] 要忽略 `app/pages/` 下某个页面，正确写法是什么？漏掉 `app/` 会怎样、你怎么得出这个结论？
- [ ] 想"忽略整个目录但放行其中一个文件"，官方给的写法是哪两行？
- [ ] `app/pages/ignore/*.vue` 里的 `ignore` 有特殊含义吗？
- [ ] 除了这个文件，还有哪三个配置项能做到类似的事？

## 下一篇

079 把这四篇讲过的目录基准收拢成一张图：`rootDir`、`srcDir`、`serverDir`、`buildDir`、`dir.*` 各自相对谁，以及 `~`、`~~`、`#build` 这些别名分别指到哪一层。

## 本文依据

- `directory-structure/nuxtignore`（标题与定义：.nuxtignore 文件让 Nuxt 在构建阶段忽略你项目根目录（rootDir）里的文件；它遵循与 .gitignore 与 .eslintignore 相同的规范，每一行是一个 glob 模式指明应忽略哪些文件；tip 也可在 nuxt.config 里配置 ignoreOptions、ignorePrefix 与 ignore；Usage 示例五组模式与注释——app/layouts/foo.vue（ignore layout foo.vue）、app/layouts/*-ignore.vue（忽略名字以 -ignore.vue 结尾的布局文件）、app/pages/bar.vue（ignore page bar.vue）、app/pages/ignore/*.vue（ignore page inside ignore folder）、app/middleware/foo/*.js 与 !app/middleware/foo/bar.js（忽略 foo 目录下的路由中间件文件但 foo/bar.js 除外）；read-more 指向 git 的 gitignore 文档以了解规范细节）
- `api/nuxt-config`（对照用：srcDir 默认 "app" 且相对路径以 rootDir 为基准、rootDir 默认 /<rootDir> 并可被 nuxt ./my-app/ 覆盖，见 051；ignore/ignorePrefix/ignoreOptions 的默认值与适用目录，见 082）
- `directory-structure`（对照用：.nuxtignore 在 Nuxt Files 一节被定义为"用于在构建阶段忽略根目录中的文件"）
- `directory-structure/gitignore`（对照用：!.env.example 那一行的例外语法与"至少包含"清单，见 077）
