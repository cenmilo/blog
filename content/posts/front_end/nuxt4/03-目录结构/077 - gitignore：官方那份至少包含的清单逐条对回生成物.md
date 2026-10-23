---
url: /posts/c4a2e2fcca8cd2b6b3aaf361eec9dab7/
title: 077 - gitignore：官方那份"至少包含"的清单，逐条对回生成物
date: 2026-11-06T11:00:00+08:00
updated: 2026-11-06T11:00:00+08:00
author: cmdragon

summary:
  摘要：.gitignore 用来指定那些有意不被 git 跟踪的文件，官方文档 .gitignore 那一节给的不是建议而是可以直接抄的清单——"我们建议有一个 .gitignore 文件，其中至少包含以下条目"，并按注释分成四组：Nuxt dev/build 输出那一组是 .output、.data、.nuxt、.nitro、.cache 与 dist；Node 依赖那一组只有 node_modules；日志那一组是 logs 与 *.log；杂项是 .DS_Store；本地环境变量那一组是 .env、.env.* 再补一行 !.env.example。这份清单与前面几篇正好互相印证：052 与 053 各用 important 要求把 .nuxt 与 .output 加进 .gitignore（前者说是避免把开发构建输出推上仓库、后者说是避免把构建输出推上仓库），071 那一节也用 important 要求把 node_modules 加进去以免把依赖推进仓库，073 那一节要求把 .env 加进去以免把密钥推进仓库；而 .env.* 之后那行以感叹号开头的写法，作用是把 .env.example 从忽略名单里排除出去，也就是"环境变量的键清单可以提交、值不要提交"。另有一处差别值得注意：这份 gitignore 清单比 nuxt cleanup 实际删除的五项更宽，它多列了 .data、.nitro 与根级 .cache，而这一节对这些额外条目没有给出解释。

categories:
  - 前端开发

tags:
  - nuxt4
  - gitignore
  - 生成物
  - 环境变量
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`.gitignore` 谁都会写，但 Nuxt 该忽略哪些这件事，官方直接给了一份清单。这一篇把那份清单抄下来，并且把每一条对回到它到底是哪个目录、哪条命令产出的，顺便指出这份清单里那些官方没解释的名字。

定义与建议措辞。官方原话，`.gitignore` 文件指定那些有意不被跟踪，intentionally untracked、git 应当忽略的文件。我们建议有一个 `.gitignore` 文件，其中至少包含以下条目。

至少包含这四个字要读准，这是下限，不是上限。

而有意不被跟踪是这份文件的语义核心，它忽略的东西，都是你随时能再生成的。这一点决定了下面第一组的性质。

那份清单。

```bash [.gitignore]
# Nuxt dev/build outputs
.output
.data
.nuxt
.nitro
.cache
dist

# Node dependencies
node_modules

# Logs
logs
*.log

# Misc
.DS_Store

# Local env files
.env
.env.*
!.env.example
```

四组注释本身就是官方的分类。逐组核对。

第一组，Nuxt 的开发与构建输出。这一组六个名字，其中两个我们已经在别的页面上见过它们各自的 important 块。

`.output` 是为生产构建时创建的目录，部署用它，跑 `nuxt build` 时整个重建，053 讲过。`.nuxt` 是开发时用来生成你的 Vue 应用的目录，跑 `nuxt dev` 时整个重建，052 讲过。`dist` 官方那一节没解释，`.data` 官方那一节没解释，`.nitro` 官方那一节没解释，`.cache` 官方那一节没解释，注意它写的是根级 `.cache`。

`.nuxt` 与 `.output` 那两节的 important 用词几乎一样，但各指一件事。052 那句，该目录应加入 `.gitignore`，以避免把开发构建输出推送到仓库。053 那句，该目录应加入 `.gitignore`，以避免把构建输出推送到仓库。两句合起来是生成物不进仓库这条原则的两次落地，和 050 那张地图上 `.nuxt` 与 `.output` 不在源码结构里是同一个判断。

官方没解释的那三个，`.data`、`.nitro`、`.cache`。官方那一节只把它们列进清单、没有说明各自是什么，文档没给定义。能确定的是两件事。

它们与 `.output` 与 `.nuxt` 同组，官方把它们归在 Nuxt dev/build outputs 之下，所以按可再生的输出对待。

`nuxt cleanup` 并不清它们，040 抄过的那份清单只有五项，构建目录，默认 `.nuxt`、`.output`、`dist`、`node_modules/.vite`、`node_modules/.cache`。

也就是说，这份 `.gitignore` 清单比 `cleanup` 的删除清单更宽，多了 `.data`、`.nitro` 与根级 `.cache`，而少了两处 `node_modules` 下的缓存。要清干净和要不提交是两件事，官方给的是两份不同的名单。

`dist`，至少有一个已知来源。049 引过官方在 `nuxt build-module` 那一节写的一句，`build-module` 命令运行 `@nuxt/module-builder`，在你的 `rootDir` 里生成一个 `dist` 目录，其中装着你的 nuxt-module 的完整构建。

所以在模块项目里，`dist` 就是构建产物，040 里 `cleanup` 也把它列进清单。051 讲 `dir.public` 时又出现过一次这个词，官方那句是 `public/` 的内容在应用生成时被复制进你的 `dist` 文件夹。两处用词一致，但都没有把它定义清楚，能确定的结论只有一个，`dist` 是输出目录，不该进仓库。

第二组与第三组，依赖与日志。

```bash [.gitignore]
node_modules
logs
*.log
```

`node_modules` 那一行有它自己那一节的 important 撑着，071 引过，这个目录应当加入你的 `.gitignore`，以避免把依赖推送进你的仓库。

这一组与第一组的理由不同，第一组是可重建，这一组是可重装，030 那张锁文件表才是重装出同样一套的依据。

`logs` 与 `*.log` 官方同样没解释，但分组注释写着 Logs。注意 `*.log` 是通配、`logs` 是目录名，两种写法各挡一类。

第四组，`.DS_Store`。一行杂项，官方注释 Misc。这是 macOS 访达写进目录的元数据文件，它属于你机器上的东西，不是项目的东西。这一句是对这个文件性质的通识说明，官方在此只写了 Misc 这个分组名。

第五组，环境变量三行，重点是那个感叹号。

```bash [.gitignore]
.env
.env.*
!.env.example
```

这三行的结构是一个忽略加例外的组合，前两行把 `.env` 和任何 `.env.xxx` 都忽略掉，第三行以 `!` 开头，作用是把 `.env.example` 从忽略名单里排除出去，这是 git 的 `.gitignore` 语义，官方页末 read-more 指向的也正是 git 官方文档。

为什么要这个例外，答案在 073 那条 important 里，这个文件应当加入你的 `.gitignore`，以避免把密钥推送进你的仓库。

于是这套写法表达的是一个很干净的工程约定。

| 文件 | 提交吗 | 内容 |
| --- | --- | --- |
| `.env`、`.env.production` 等 | 不 | 真实的值，密钥、连接串 |
| `.env.example` | 要 | 这份项目需要哪些键的清单，值留空或写占位 |

换句话说，`.env.example` 是给同事看的接口说明，而它确实该跟 `nuxt.config` 一起进仓库，因为 073 讲过，运行时的值由 `NUXT_*` 环境变量注入，018 那条硬规则，别人要知道注入哪些键，只能看这份示例。

一处容易混，`.gitignore` 与 `.nuxtignore`。两个文件名字像、管的东西完全不同。

| | `.gitignore` | `.nuxtignore` |
| --- | --- | --- |
| 谁在用 | git | Nuxt |
| 作用 | 不被仓库跟踪 | 在构建阶段忽略根目录里的文件，050 原话 |
| 时机 | 你提交时 | 你构建时 |

下一篇 078 就专门讲 `.nuxtignore`，它有个坑，路径基准与写法。

## 自查

- [ ] 官方的措辞是至少包含，它的第一组注释叫什么？
- [ ] `.nuxt` 与 `.output` 各自那句 important 是怎么说的？为什么它们能再生？
- [ ] `nuxt cleanup` 的清单和这份 `.gitignore` 清单差在哪几项？
- [ ] `dist` 至少有一个官方明确说明的来源，是哪个命令产出的？
- [ ] `!.env.example` 那一行起什么作用？为什么它值得单独提交？
- [ ] `.gitignore` 与 `.nuxtignore` 分别由谁在什么时机使用？

## 下一篇

078 讲 `.nuxtignore`：官方给的基准是 `rootDir`、语法是 gitignore 规范，但那句最关键的警告是——这里的模式必须以 `app/` 开头，否则它不会匹配任何东西（051 讲的 `srcDir` 默认值就是原因）。
