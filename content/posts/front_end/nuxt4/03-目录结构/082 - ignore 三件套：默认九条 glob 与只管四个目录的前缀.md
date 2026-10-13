---
url: /posts/3c98122f3e31b4bb42b579b690de8ac7/
title: 082 - ignore 三件套：默认那九条 glob，和只管四个目录的前缀
date: 2026-11-08T10:00:00+08:00
updated: 2026-11-08T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方给了三个互补的配置项来做"让 Nuxt 看不见某些文件"这件事。ignore 比 ignorePrefix 更可定制：所有匹配 ignore 数组里那些 glob 模式的文件在构建时都会被忽略，类型为 array，默认九条分别是 **/*.stories.{js,cts,mts,ts,jsx,tsx}、**/*.{spec,test}.{js,cts,mts,ts,jsx,tsx}、**/*.d.{cts,mts,ts}、**/*.d.vue.{cts,mts,ts}、**/.{pnpm-store,vercel,netlify,output,git,cache,data}、**/*.sock、.nuxt/analyze、.nuxt 与 **/-*.*，其中第五条那一串花括号里列的正是各种工具目录、而 .nuxt 本身就出现在名单里。ignorePrefix 的作用域则明确得多：只有 app/pages/、app/layouts/、app/middleware/ 与 public/ 这四个目录里的文件，如果文件名以它指定的前缀开头就会在构建过程中被忽略，官方说明其意图是防止某些文件被内置应用处理或被提供出去，默认值是 "-" 也就是忽略任何以 - 开头的文件。ignoreOptions 是把选项直接传给 node-ignore（Nuxt 用来忽略文件的实现），官方给的示例是 ignoreOptions: { ignorecase: false }。

categories:
  - 前端开发

tags:
  - nuxt4
  - ignore
  - glob
  - 构建
  - 配置

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


078 末尾我留了三个名字。这一篇兑现它们：**`ignore`、`ignorePrefix`、`ignoreOptions`**——三份配置，作用域完全不同。

**先给结论：默认那九条 glob 里，藏着一份"哪些目录不属于你的应用"的官方名单。**

## 分工表：先看清三者不是一回事

| 配置项 | 官方措辞 | 作用域 | 默认值 |
| --- | --- | --- | --- |
| `ignore` | "比 `ignorePrefix` 更可定制：所有匹配 `ignore` 数组里 glob 模式的文件，**在构建时都会被忽略**" | 全局（glob 决定） | 九条 glob（见下） |
| `ignorePrefix` | **只有四个目录**里的文件按文件名前缀忽略 | `app/pages/`、`app/layouts/`、`app/middleware/`、`public/` | `"-"` |
| `ignoreOptions` | 把选项直接传给 `node-ignore`（Nuxt 用来忽略文件的实现） | 影响忽略实现本身 | （无默认值，示例给了一条） |
| `.nuxtignore` 文件 | 构建阶段忽略 `rootDir` 里的文件，一行一个 glob | 见 078 | —— |

**`ignore` 与 `.nuxtignore` 是同一族（数组 vs 文件），`ignorePrefix` 是另一族（就地打标），`ignoreOptions` 是第三个维度（改规则引擎的行为）。**

## 默认那九条，逐条读

官方给的 `ignore` 默认值（类型 `array`）：

```json
[
  "**/*.stories.{js,cts,mts,ts,jsx,tsx}",
  "**/*.{spec,test}.{js,cts,mts,ts,jsx,tsx}",
  "**/*.d.{cts,mts,ts}",
  "**/*.d.vue.{cts,mts,ts}",
  "**/.{pnpm-store,vercel,netlify,output,git,cache,data}",
  "**/*.sock",
  ".nuxt/analyze",
  ".nuxt",
  "**/-*.*"
]
```

**分四类看：**

**一、测试与故事文件（第 1、2 条）。** `*.stories.*` 与 `*.spec.*`/`*.test.*`——**这些不该被注册成组件或页面**。

**二、类型声明（第 3、4 条）。** `**/*.d.{cts,mts,ts}` 与 `**/*.d.vue.{cts,mts,ts}`——**`.d.` 这种纯类型文件不进构建。**

**三、工具与生成目录（第 5、7、8 条）。第 5 条最有信息量：**

```
**/.{pnpm-store,vercel,netlify,output,git,cache,data}
```

**它把 `.output`、`.cache`、`.data`、`.git`、`pnpm-store`、`vercel`、`netlify` 一次列进同一个花括号。** 对照 077 那份 `.gitignore` 清单——**同样的名字（`.output`、`.data`、`.cache`）在两份名单里都出现**：一份是"别提交"，这一份是"**别扫描**"。

**第 7、8 条则是 `.nuxt/analyze` 与 `.nuxt` 本身——框架不会去扫描自己的生成目录。**

**四、两类杂项（第 6、9 条）。** `**/*.sock` 是套接字文件；`**/-*.*` 则是"**任何路径下、文件名以 `-` 开头**的文件"。

**第 9 条与下面 `ignorePrefix` 的默认值 `-` 是同一意图的两份表达**——差别就在作用域：`**/-*.*` 是全局 glob，而 `ignorePrefix` 只写那四个目录（官方限定，见下节）。

## `ignorePrefix`：作用域写死在四个目录

官方这一句的限定词很关键，我完整抄：

> **`app/pages/`、`app/layouts/`、`app/middleware/` 与 `public/` 目录里的任何文件**，如果它的文件名以 `ignorePrefix` 指定的前缀开头，就会在构建过程中被忽略。**其意图是防止某些文件在构建后的应用里被处理或被提供出去。默认情况下 `ignorePrefix` 被设为 `'-'`，也就是忽略任何以 `-` 开头的文件。**

**三个可执行结论：**

1. **它只管这四个目录。** 也就是说，**`app/components/-Draft.vue` 不会因为这个前缀被忽略**——官方列的四个目录里没有 `components/`（这一句是我按那份四目录清单得到的判断，官方没有额外说明为什么不含它）。要排除组件，用 `ignore` 数组或 `.nuxtignore`（078）。
2. **`public/` 也在名单里**——这条与 064 呼应：**`public/` 是"原样提供、不被构建修改"的位置，所以那里"不想被提供出去"的文件，只能靠文件名前缀或 ignore 规则挡掉。**
3. **默认前缀是 `-`**，所以 `-draft.vue`、`-helper.ts` 这类命名是**官方支持的"就地禁用"手法**。

## `ignoreOptions`：改的是规则引擎

> 把选项直接传给 `node-ignore`（Nuxt 用来忽略文件的实现）。

官方给的示例只有一条：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  ignoreOptions: {
    ignorecase: false,
  },
})
```

**这一条能推出一件有用的事：默认的忽略匹配是不区分大小写的**（否则不需要给你一条把它关掉的示例）。**所以 `Ignore.vue` 与 `ignore.vue` 在默认设置下会命中同一条规则。**（官方在这里没有明说默认值，只给了 `ignorecase: false` 这个例子，**这一句是我按示例方向推的判断**。）

## 三个入口怎么选：一张决策表

| 你要排除的东西 | 推荐做法 | 依据 |
| --- | --- | --- |
| 某个页面/布局/中间件暂时不生效 | 文件名加 `-` 前缀 | `ignorePrefix` 默认值与作用域 |
| `public/` 里某个文件不要被提供 | 加 `-` 前缀，或 `ignore` 一条 glob | 四个目录含 `public/` |
| 某类文件全局别扫（例如 `**/*.fixture.ts`） | `ignore` 数组 | "所有匹配 glob 模式的文件在构建时都会被忽略" |
| 只想排除 `rootDir` 下某几个具体文件 | `.nuxtignore` 一行一个模式 | 078 |
| 大小写匹配行为 | `ignoreOptions` | 传给 `node-ignore` |

**注意最后一行的分层：`.nuxtignore` 是文件、`ignore` 是数组**，两者都能表达 glob，**差别是前者随项目单独一个文件、后者写在 `nuxt.config` 里**（078 那条 tip 就是把这三个键并列给你的）。

## 自查

- [ ] 默认那九条 `ignore` 里，哪一条同时挡住了 `.output`、`.cache` 与 `.data`？它和 077 那份 `.gitignore` 清单是什么关系？
- [ ] `.nuxt` 为什么也出现在 `ignore` 默认值里？
- [ ] `ignorePrefix` 默认是什么？它管几个目录？`components/` 在不在里面？
- [ ] 想让 `app/components/-Draft.vue` 不被注册，改 `ignorePrefix` 有用吗？该用什么？
- [ ] `ignoreOptions` 传给谁？官方那条示例暗示了默认是什么行为？
- [ ] "暂时让这个页面不生效"最便宜的做法是什么？

## 下一篇

083 进入生成物排错：`debug` 与 `logLevel` 这两个开关各自打开什么、CI 里日志级别被自动降成什么，以及为什么 `debug: true` 的输出只覆盖服务端钩子。

## 本文依据

- `api/nuxt-config`（ignore：比 ignorePrefix 更可定制，所有匹配 ignore 数组内 glob 模式的文件将在构建时被忽略，类型 array，默认九条 "**/*.stories.{js,cts,mts,ts,jsx,tsx}"、"**/*.{spec,test}.{js,cts,mts,ts,jsx,tsx}"、"**/*.d.{cts,mts,ts}"、"**/*.d.vue.{cts,mts,ts}"、"**/.{pnpm-store,vercel,netlify,output,git,cache,data}"、"**/*.sock"、".nuxt/analyze"、".nuxt"、"**/-*.*"。ignoreOptions：把选项直接传给 node-ignore（Nuxt 用来忽略文件的实现），See 指向 node-ignore 仓库，示例 ignoreOptions: { ignorecase: false }。ignorePrefix：app/pages/、app/layouts/、app/middleware/ 与 public/ 目录里的任何文件，若其文件名以 ignorePrefix 指定的前缀开头则在构建过程中被忽略，其意图是防止某些文件在构建后的应用里被处理或被提供出去，默认 "-" 即忽略任何以 - 开头的文件，类型 string）
- `directory-structure/nuxtignore`（对照用：.nuxtignore 在构建阶段以 rootDir 为基准忽略文件、每行一个 glob、并 tip 指出也可在 nuxt.config 配置 ignoreOptions/ignorePrefix/ignore，见 078）
- `directory-structure/gitignore`（对照用：官方"至少包含"清单里的 .output/.data/.cache/dist 等条目，见 077）
- `directory-structure/public`（对照用：public/ 里的文件在根路径提供且不被构建过程修改、适合必须保住名字的文件，见 064）
- `directory-structure/app/components`（对照用：components 目录的自动注册与命名规则、以及"目录项指定 pattern 时 extensions 无效"，见 062）
