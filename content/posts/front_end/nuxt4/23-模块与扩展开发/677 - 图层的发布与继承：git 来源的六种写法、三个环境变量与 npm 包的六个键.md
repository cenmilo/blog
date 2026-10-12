---
url: /posts/4480fdf3ad39e378b5e54618c4b5887b/
title: 图层的发布与继承：git 来源的六种写法、三个环境变量与 npm 包的六个键
date: 2027-05-25T11:00:00+08:00
updated: 2027-05-25T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`extends` 一个字符串数组就能继承别的层，而这一页把它的三种来源与各自的坑写全了。**git 来源有六种写法**（`github:用户/仓库`、带子目录 `github:.../base`、带分支 `#dev`、带标签 `#v1.0.0`、`gitlab:`、`bitbucket:`），**未指定分支时会 clone `main`**；私有仓库要 `GIGET_AUTH=<token>`，自建实例要 `GIGET_GITHUB_URL` 或 `GIGET_GITLAB_URL`；git 层若要装它自己的依赖，得显式写 `install: true`。**最该提前知道的一条警告是**：以远程层方式引入的依赖会被装到 `node_modules/.c12/层名/node_modules/`，**你的包管理器看不见它**——所以官方推荐把层发成 npm 包（公共或私有），或在 `package.json` 里直接用 git URL 作依赖。走 npm 时**层的 `package.json` 要填六个键**，其中 **`main` 指向 `./nuxt.config.ts`**，且**层里 import 到的每个依赖都必须显式列进 `dependencies`**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 图层
  - extends
  - npm发布
  - git来源
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

把层共享出去有两条路：一个 git 仓库，或一个 npm 包。两条都能用，但**它们在"依赖装在哪"上的差异会让你后期的 lint、测试和 CI 出现解释不了的现象**。

## `extends` 的三种来源

入门页给的最小形式是三种混在一起：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    // Extend from a local layer
    '../base',
    // Extend from an installed npm package
    '@my-themes/awesome',
    // Extend from a git repository
    'github:my-themes/awesome#v1',
  ],
})
```

三条注释就是三种来源的分类：**本地层（相对路径）**、**已安装的 npm 包**、**git 仓库**。npm 那条还带 scope，写 `@scope/moduleName` 或裸 `moduleName` 都行。

底层是谁也说了：**Nuxt 用 unjs/c12 与 unjs/giget 来继承远程层**，并让你去查它们的文档看全部可用选项。

## git 来源的六种写法

作者页把 git 形态展开成一份清单：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    // GitHub Remote Source
    'github:username/repoName',
    // GitHub Remote Source within /base directory
    'github:username/repoName/base',
    // GitHub Remote Source from dev branch
    'github:username/repoName#dev',
    // GitHub Remote Source from v1.0.0 tag
    'github:username/repoName#v1.0.0',
    // GitLab Remote Source example
    'gitlab:username/repoName',
    // Bitbucket Remote Source example
    'bitbucket:username/repoName',
  ],
})
```

三个可组合的后缀要分开记：**斜杠接子目录**（`/base`，仓库里不是根的位置）、**`#` 接分支或标签**（`#dev`、`#v1.0.0`）、**前缀换平台**（`gitlab:`、`bitbucket:`）。

一条默认值写在 `::note` 里：**如果不指定分支，这会 clone `main`**。

**这有实际后果**：不写 `#v1.0.0` 而依赖默认分支，等于把你的构建钉在仓库的 HEAD 上。

## 私有仓库与自建实例

| 需求 | 做法 |
| --- | --- |
| 扩展私有远程源 | 加环境变量 `GIGET_AUTH=<token>` 提供 token |
| 扩展自建的 GitHub 实例 | `GIGET_GITHUB_URL=<url>` |
| 扩展自建的 GitLab 实例 | `GIGET_GITLAB_URL=<url>` |
| 不想用环境变量 | **在 `nuxt.config` 里用 `auth` 选项直接配置**（链到 c12 的文档） |

入门页也给了一条**写在数组项里的等价形式**——用元组把 token 直接传：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    // per layer configuration
    ['github:my-themes/private-awesome', { auth: process.env.GITHUB_TOKEN }],
  ],
})
```

两种入口（环境变量与 `auth` 选项）解决同一件事，**页面没有说明同时给出时谁优先**。

## 最该提前知道的那条警告

作者页有一条 `::warning`，讲的是一个后期很难查的限制：

> Bear in mind that if you are extending a remote source as a layer, you will not be able to access its dependencies outside of Nuxt.

> For example, if the remote layer depends on an eslint plugin, this will not be usable in your eslint config. That is because these dependencies will be located in a special location (`node_modules/.c12/layer_name/node_modules/`) that is not accessible to your package manager.

拆开是三层：**远程层的依赖你在 Nuxt 之外访问不到**；**举的例子是"层依赖了一个 eslint 插件，那在你的 eslint 配置里就用不了"**；**原因是这些依赖被装在 `node_modules/.c12/层名/node_modules/` 这个特殊位置，你的包管理器不可见**。

官方给出的对策在那条 `::note` 里，两条路：**推荐把层的内容发布成 npm 包（公共或私有，例如通过 GitHub Packages），而不是依赖远程层**；或者**把远程 git URL 直接作为依赖加进 `package.json`**（链到 npm 文档的 "git urls as dependencies" 一节）。

也就是说：**"从 git 直接拉层"适合省事，代价是层带的工具链依赖会脱离你的包管理器视野。**

还有一条与 git 来源相关的开关：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    ['github:username/repoName', { install: true }],
  ],
})
```

**`install: true` 的作用是让这个 git 层的 npm 依赖被安装**——反过来说，**默认不装**，这与上一条警告是同一件事的另一面。

## 发成 npm 包要做的准备

作者页的顺序是三句：**先确认模块已发布到 npm 并被用户装成 devDependency**，然后就能用包名 extends：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    // Node Module with scope
    '@scope/moduleName',
    // or just the module name
    'moduleName',
  ],
})
```

**注意"装成 devDependency"这句是发布方的使用要求**（层不参与运行时产物），与模块本身要进 `dependencies` 的常规做法不同。

要让文件被正确打进包，`package.json` 得填对这些键：

```json [package.json]
{
  "name": "my-theme",
  "version": "1.0.0",
  "type": "module",
  "main": "./nuxt.config.ts",
  "dependencies": {},
  "devDependencies": {
    "nuxt": "^3.0.0"
  }
}
```

六个键各有作用：**`name`／`version` 是包身份**、**`type: "module"`**、**`main` 指向 `./nuxt.config.ts`（这就是"有一个 nuxt.config.ts 就说明它是层"这件事在包里的表达方式）**、**`dependencies` 与 `devDependencies` 的分界**。

分界由一条 `::important` 写死：

> Make sure any dependency imported in the layer is **explicitly added** to the `dependencies`. The `nuxt` dependency, and anything only used for testing the layer before publishing, should remain in the `devDependencies` field.

也就是三个规则：**层里 import 到的每个依赖都要显式进 `dependencies`**、**`nuxt` 本身留在 `devDependencies`**、**发布前只用于测试的东西也留在 `devDependencies`**。

私有发布还有一条：**把层作为私有 npm 包发布时，要先 `login` 完成 npm 认证才能下载这个包**（这条出现在 `::important` 里，措辞是"确保你 log in"）。

## 两种发布方式怎么选

| 你的情况 | 选 |
| --- | --- |
| 层只给自己几个仓库用、且不想发版 | git 来源（**记得钉分支或标签**） |
| 层的依赖需要被你的工具链看到（eslint 插件、构建工具） | **npm 包**，或把 git URL 写成 `package.json` 的依赖 |
| 要控制"层的依赖到底装不装" | git 来源时靠 `install: true`；npm 包时靠包管理器本身 |
| 私有分发 | 私有 npm 包（**需要 npm 登录**）或带 token 的 git（`GIGET_AUTH` / `auth`） |

这一页没写的：`extends` 指向一个 monorepo 内部目录时的最佳实践、git 层被 clone 到哪个目录（**只有那条 `.c12` 路径的侧面提示**）、`install: true` 用哪种包管理器安装，以及层的 `version` 与 `nuxt` 兼容范围之间有没有校验（`meta.compatibility` 是模块那套机制，页面没有把它接到层上）。

下一篇讲优先级：一个资源被多层同时定义时，谁赢。

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/layers
- https://nuxt.com/docs/4.x/getting-started/layers
