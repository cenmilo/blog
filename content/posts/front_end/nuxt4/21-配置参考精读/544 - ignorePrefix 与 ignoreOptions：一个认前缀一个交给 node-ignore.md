---
url: /posts/89607e2d93fcc9989f56c48b4e6d46f7/
title: ignorePrefix 与 ignoreOptions：一个认前缀一个交给 node-ignore
date: 2027-04-11T10:00:00+08:00
updated: 2027-04-11T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`ignorePrefix` 的作用面被官方点名在**四个目录**：**"Any file in `app/pages/`, `app/layouts/`, `app/middleware/`, and `public/` directories will be ignored during the build process if its filename starts with the prefix specified by `ignorePrefix`."**，目的句是"防止某些文件被处理或被构建后的应用服务"，默认值 **`"-"`**（也就是以 `-` 开头的文件一律忽略）。`ignoreOptions` 则完全不同：**"Pass options directly to `node-ignore` (which is used by Nuxt to ignore files)."**——它是把选项原样交给 node-ignore 这个库，官方示例只有一个：`ignoreOptions: { ignorecase: false }`。第三个入口是项目根的 `.nuxtignore` 文件，官方在那一页直接把三者并列：你**也可以**在 `nuxt.config` 里配 `ignoreOptions`、`ignorePrefix` 和 `ignore`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建
  - ignorePrefix
  - node-ignore
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`_` 还是 `-` 开头能排除文件？这个问题看着一样，答案分两条：一条是"某个前缀"，一条是"glob 模式"，第三条干脆写在项目根的一个文件里。

## ignorePrefix：只认前缀，只在这四个目录

整节一句话，但每个限定都有用：

> Any file in `app/pages/`, `app/layouts/`, `app/middleware/`, and `public/` directories will be ignored during the build process if its filename starts with the prefix specified by `ignorePrefix`. This is intended to prevent certain files from being processed or served in the built application. By default, the `ignorePrefix` is set to '-', ignoring any files starting with '-'.

- **类型**：`string`
- **默认**：`"-"`

拆成四个限定：

| 限定 | 官方措辞 | 意味着什么 |
| --- | --- | --- |
| 目录范围 | 点名 `app/pages/`、`app/layouts/`、`app/middleware/`、`public/` | **只有这四个目录受前缀规则影响**；其它位置要用 `ignore` |
| 判定方式 | its **filename** starts with the prefix | 看的是**文件名开头**，不是路径段、不是内容 |
| 时机 | ignored **during the build process** | 影响构建产物，不是运行时行为 |
| 目的 | prevent certain files from being processed or **served** in the built application | 两类：不被处理（页面/布局/中间件注册）、不被服务（`public` 里的文件不出现在产物里） |

最后一行的"or served"值得单独看：`public/` 里以 `-` 开头的文件默认**不会被服务**，这与前三个目录的"不被注册"是两种结果，但都由同一条前缀规则触发。

**注意前缀默认是 `-` 而不是 `_`。** 很多人凭习惯用下划线开头想排除文件，那条规则在这里不成立。

## ignoreOptions：把选项交给库

这节的文字很少，但有一句关键定性：**"Pass options directly to `node-ignore` (which is used by Nuxt to ignore files)."**

括号里那句是全套文档里唯一一处说明**"Nuxt 判断忽略用的是什么实现"**——**node-ignore 这个库**。官方给的示例只有一行配置：

```ts
export default defineNuxtConfig({
  ignoreOptions: {
    ignorecase: false,
  },
})
```

`ignorecase: false` 的效果是**关掉大小写不敏感**（默认行为是把它当 `.gitignore` 那样不区分大小写处理，库的这一项就叫 `ignorecase`）。

两个信息缺口要说清：**这一节没有列 `**Type**`，也没有列默认值**（属于那份"约二十三个只有类型或什么都没有"的条目之一），而且**可用选项的完整清单官方不在这页给**——它只留了一个 **See** 指向 node-ignore 仓库。要设别的选项，得去那库的文档查，Nuxt 侧没有任何一个"已验证可用"的白名单。

## 第三个入口：`.nuxtignore`

`.nuxtignore` 那一页把这三者并列：**"You can also configure `ignoreOptions`, `ignorePrefix` and `ignore` in your `nuxt.config` file."**

而文件本身是这么定义的：它告诉 Nuxt 在**构建阶段**忽略 `rootDir` 里的文件，**"It is subject to the same specification as `.gitignore` and `.eslintignore` files, in which each line is a glob pattern indicating which files should be ignored."**

官方给的示例正好把"前缀规则"与"模式规则"的区别演出来了：

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

三处值得注意：路径**写的是从 `rootDir` 起的完整路径**（`app/layouts/foo.vue`，不是 `layouts/foo.vue`）；**支持 `!` 取反**（最后一行排除一个文件不忽略）；示例里出现了 `*-ignore.vue` 这种"以 `-ignore` 结尾"的写法——这与 `ignorePrefix` 的"以 `-` 开头"是**不同方向的规则**，别混用。

## 三个入口怎么选

按文档能给出的分工：

| 你要做的事 | 入口 |
| --- | --- |
| 排除某个 glob 形态（后缀、目录、生成物） | `ignore`（数组，默认九条） |
| 让某个前缀的文件在那四个目录里不生效 | `ignorePrefix`（一个字符串） |
| 改匹配行为本身（比如大小写敏感） | `ignoreOptions`（交给 node-ignore） |
| 按 gitignore 语法逐行列、还要取反 | `.nuxtignore` 文件 |

`.nuxtignore` 与 `ignore` 都在"匹配模式"这一层，**官方没有说明两者叠加时的优先或去重规则**；`!` 取反只在 `.nuxtignore` 那页作为 gitignore 规范提到，`ignore` 数组里能否用取反，文档没写。

## 自查

- `ignorePrefix` 是 `string`、默认 `"-"`，判定的是**文件名开头**，生效范围是官方点名的四个目录（`app/pages/`、`app/layouts/`、`app/middleware/`、`public/`），时机是构建过程，目的含"不被处理或不被服务"。
- 前缀是 `-` 不是 `_`；`public` 里以 `-` 开头的文件默认不会被服务。
- `ignoreOptions` 把选项原样交给 node-ignore，官方只示范 `ignorecase: false`；该节没有类型与默认值，可用选项清单指向那个库。
- `.nuxtignore` 与 `.gitignore` 同一规范、每行一个 glob、支持 `!` 取反，路径从 `rootDir` 写起。
- 三个 `nuxt.config` 入口与 `.nuxtignore` 之间的叠加优先规则，官方未说明。

## 下一篇

下一篇 545 读 `watch` 与 `watchers`：前者定义"改了哪些东西就重启开发服务器"、字符串按 `srcDir` 算，后者分三块——chokidar 两个默认 `true` 的选项、`rewatchOnRawEvents` 这个没有类型的事件数组、以及 webpack 的 `watchOptions.aggregateTimeout` 默认 `1000`。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## ignorePrefix` 整节：四个目录点名句、filename starts with the prefix、prevent being processed or served、默认 `'-'` 与 Type/Default；`## ignoreOptions` 整节：node-ignore 定位句、See 链接与 `ignorecase: false` 示例，该节无 Type/Default；`## ignore` 的默认九条作为对照）
- https://nuxt.com/docs/4.x/directory-structure/nuxtignore（rootDir 与构建阶段忽略的定义句、与 `.gitignore`/`.eslintignore` 同一规范且每行一个 glob、并列三个配置键的 `::tip`、Usage 示例全文含 `!` 取反与从 rootDir 起写的路径）
