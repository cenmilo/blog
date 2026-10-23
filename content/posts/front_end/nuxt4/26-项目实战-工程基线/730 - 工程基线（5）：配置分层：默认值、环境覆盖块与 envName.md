---
url: /posts/1ada736ce467d9006c014a15c88dea40/
title: 730 - 工程基线（5）：配置分层：默认值、环境覆盖块与 envName
date: 2026-10-18T13:00:00+08:00
updated: 2026-10-18T13:00:00+08:00
author: cmdragon

summary:
  摘要：配置不是一个文件说了算。项目根里那个没人写过的 `.nuxtrc`、`nuxt.config.ts` 里 `$production` 与 `$env.staging` 这几块、命令上的 `--envName`，三者有固定顺序，而且都发生在构建期。这一站用五次生成把顺序量出来，再验一件事：根目录放一份 `vite.config.ts`，它一次都没被读。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 配置分层
  - envName
  - nuxtrc
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先认一个你没写过的文件

正式项目的根目录里就躺着这么一个文件，打开只有一行：

```bash [.nuxtrc]
setups.@nuxt/test-utils="4.3.3"
```

没人打算配置什么，它自己出现了。官方讲 `.nuxtrc` 的那一节给了原因：Nuxt 会自动往里加一个 `setups` 段，记录模块的安装与升级状态，**这一段是内部用的，不要手改**。

但这个文件本身是真的会被读的，而且它有位置上的层级。把这一层与 `nuxt.config.ts`、以及配置文件里那几个 `$` 开头的块放在一起，就是这一站要量清楚的东西：**谁盖谁、什么时候选、选错了会怎样。**

## 顺序：全局 `.nuxtrc` → 项目 `.nuxtrc` → `nuxt.config`

官方写了两句话，合起来是一条完整链子：项目里如果有 `.nuxtrc`，`nuxt.config` 的属性会盖过它；而在用户目录下还有一份全局 `.nuxtrc`（macOS/Linux 是 `~/.nuxtrc`，Windows 是 `C:\Users\{username}\.nuxtrc`），它给本机所有 Nuxt 项目定默认值，**项目级 `.nuxtrc` 盖过全局，`nuxt.config` 盖过前两者**。

`.nuxtrc` 用的是扁平写法（底层是 `unjs/rc9`），官方给的例子就是四行：

```bash [.nuxtrc]
# Disable SSR
ssr=false

# Configuration for `@nuxt/devtools`
devtools.enabled=true

# Add Nuxt modules
modules[]=@nuxt/image
modules[]=nuxt-security

# Module setups (automatically added by Nuxt)
setups.@nuxt/test-utils="3.23.0"
```

点号表示嵌套、`[]` 表示往数组里追加一项。那一节还有一句 tip：要做更复杂的配置，请用 `nuxt.config`——扁平文件适合放"一两行开关"，不适合放函数。

这条链子可以当场验。在最小工程里只写 `.nuxtrc` 一行：

```bash
app.head.title=FROM-RC
```

```text
npx nuxt generate          →  <title>FROM-RC</title>
```

再把同一个位置写进 `nuxt.config.ts`（根上的 `app.head.title` 设成另一个值），跑一次没有对应覆盖块的构建：

```text
npx nuxt generate --envName qa   →  <title>FROM-CONFIG</title>
```

`.nuxtrc` 那行还在，但它的值没出现。顺序不是记在脑子里的，是能这么验的。（全局那一级没有在这台机器上验证——不会去动用户目录里的文件，所以那一层只按文档讲。）

## 环境覆盖块：`$production`、`$development`、`$env.<名字>`

`nuxt.config.ts` 里可以用 `$` 开头的键写"只在某个环境生效"的配置，官方给的形状就是这样：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  $production: {
    routeRules: {
      '/**': { isr: true },
    },
  },
  $development: {
    //
  },
  $env: {
    staging: {
      //
    },
  },
})
```

选哪一个环境靠命令旗标：`nuxt build --envName staging`。这套覆盖的机制来自 Nuxt 配置文件解析器 `c12` 的"按环境配置"，那一节直接把读者指过去。还有一句给写层（layer）的人：可以用 `$meta` 键放元数据，给你或你这套层的消费者读。

`--envName` 出现在五条命令的选项表里——`build`、`dev`、`preview`、`generate`、`prepare`，五条的描述是同一句话：解析配置覆盖时用的环境名，**默认在构建时是 `production`、跑开发服务时是 `development`**。注意这五行 `Default` 那一栏都是空的，默认值写在描述文字里而不是表格里，所以只看表会以为没有默认。

把四个块同时写在一个最小工程里（根 `app.head.title` 与 `$production`、`$development`、`$env.staging` 各给一个不同的值），再跑五次：

| 怎么跑 | HTML 里的 title | 读出来的规则 |
| --- | --- | --- |
| `npx nuxt generate` | `FROM-PRODUCTION` | 不带旗标就按 `production` 解析 |
| `npx nuxt generate --envName staging` | `FROM-STAGING` | 选中 `$env.staging`，`$production` 那一块没参与 |
| `npx nuxt generate --envName development` | `FROM-DEVELOPMENT` | 名字直接对上 `$development` 这一块 |
| `npx nuxt generate --envName qa` | `FROM-CONFIG` | 没有叫 `qa` 的块，于是用根上那份 |
| `npx nuxt generate --envName production` | `FROM-PRODUCTION` | 配置里还写了一块拼成 `$prodction`，**它被静默忽略** |

最后一行是这一站最要记住的：**名字写错不会报错、不会警告，退出码还是 0**，构建日志里连一句提示都没有（那趟日志里只有两条与依赖解析有关的 `WARN`，和配置无关）。它的表现只是"这块配置没生效"。所以加一块覆盖配置之后，要跑一次**带上对应 `--envName`** 的构建，并在产物里 grep 那个值——下一节就是这么做的。

## envName 是构建期的事，一份产物只属于一个环境

把同一份配置用 `--envName staging` 构建（不是 generate，是普通 build），然后去产物里找那个值：

```text
npx nuxt build --envName staging        rc=0，Σ Total size: 1.96 MB (491 kB gzip)
grep -rl "FROM-STAGING" .output/server  → .output/server/chunks/routes/renderer.mjs
```

值已经烤进服务端代码里了。**这意味着运行期改环境变量换不掉选中的覆盖块**：要 `production` 与 `staging` 两套，就构建两次、交两份产物。

代码里可以把当前环境名读回来。`import.meta.envName` 从 v4.5 起提供，包含 `--envName` 传进去的自定义值；把它印到页面上再预渲染，读回来的是 `staging`。但同一个表达式写在 `nuxt.config.ts` 自己身上，量到的值是 `undefined`——**配置文件里判断不了当前环境名，要判断就到应用代码里读。**

程序化启动 Nuxt 也有一位同名参数：`loadNuxt` 的选项 `envName`，类型是 `string | false`、默认 `undefined`，文档明写它**优先于 `nuxt.config` 里设的 `envName`**——也就是说配置文件里可以定一个默认环境名，命令行再盖它。装好的类型包里对这一位还有一句更完整的说明：它的值先取显式传进来的（例如 `nuxt --envName`），否则**在 dev 模式下取 `'development'`，其它情况取 `'production'`**，并且"以 `import.meta.envName` 暴露给应用代码"。上面那四次生成的读数正好是这句话的两半：不带旗标的 `generate` 落到了 `production`，而 dev 那一路没法在这台机器上验（它要一个常驻的开发服务）。

还有一条对照，容易和这套机制混：讲 `app.baseURL` 那节明写它可以在运行期用 `NUXT_APP_BASE_URL` 设。两条通道分工不同——`$env` 与 `--envName` 选的是**构建时读哪一份配置**，`NUXT_` 前缀那一套改的是**运行时那份配置里的值**。后者正是下一篇要接住的 `runtimeConfig`。

正式项目里这份配置已经按这个结构写了，可以对着看：`$production` 放预渲染开关与整张 `routeRules` 表，`$env.staging` 只把标题换成 `Substrate (staging)`，`$development` 那条 `devtools` 与根上同值（删不删都不改变结果，留着只是把意图写出来）。另外项目里还有一行不属于这套机制的写法：用 `process.env` 在配置求值时拼出一段 `experimental`——它同样发生在构建期（读的是跑构建那台机器的变量），但要靠展开运算符手写，读代码的人得往下翻才看得到，**能用 `$env` 块说清的事就别用 spread**。

## 唯一真源：那四个划了删除线的文件

官方有一张表专门讲外部配置文件，四行都带删除线：

| 名字 | 配置文件 | 怎么配 |
| --- | --- | --- |
| Nitro | ~~`nitro.config.ts`~~ | 用 `nuxt.config` 里的 `nitro` 键 |
| PostCSS | ~~`postcss.config.js`~~ | 用 `nuxt.config` 里的 `postcss` 键 |
| Vite | ~~`vite.config.ts`~~ | 用 `nuxt.config` 里的 `vite` 键 |
| webpack | ~~`webpack.config.ts`~~ | 用 `nuxt.config` 里的 `webpack` 键 |

这一条值得自己验一次，而且要带对照组。在 `nuxt.config.ts` 的 `vite.plugins` 里放一个"构建开始时打一行字"的插件，同时在项目根再放一份 `vite.config.ts`，里面放同样一个插件、打另一行字：

```text
PROBE-CONFIG-KEY   出现 2 次   （客户端与服务端各一趟构建）
PROBE-EXTERNAL-FILE 出现 0 次
构建 rc=0，没有任何关于 vite.config.ts 的提示
```

**根目录那份 `vite.config.ts` 一次都没被读，而框架不提一句。** 给 Vite 传东西只有 `nuxt.config` 里那一位。

但"所有配置都并进 `nuxt.config`"是误记——紧挨着还有一张表，列的仍是独立文件：`tsconfig.json`、`eslint.config.js`、`prettier.config.js`、`stylelint.config.js`、`tailwind.config.js`、`vitest.config.ts`。划删除线的只有构建工具那四个。

要给 Vue 传选项也走 `nuxt.config`：`vite.vue` 对应 `@vitejs/plugin-vue`、`vite.vueJsx` 对应 `@vitejs/plugin-vue-jsx`、webpack 那边用 `webpack.loaders.vue`，而 `vue.propsDestructure: true` 这一位不分打包器都生效（`reactivityTransform` 从 Nuxt 3.9、Vue 3.4 起已经移给 Vue Macros，不再由这里配）。

最后一句关于生效时机：Nuxt 会盯着主配置文件、`.env`、`.nuxtignore` 与 `.nuxtrc`，检测到改动就整站重启（不是热更新）。所以开发时改一行 `.nuxtrc` 看到页面白一下，是正常的。

## 这一步的自查清单

1. 根目录有 `.nuxtrc` 吗？只有 `setups.*` 那几行就是 Nuxt 自己写的，别手改；要加配置就加到 `nuxt.config.ts`。
2. `nuxt.config.ts` 里 `$production`、`$development`、`$env.<名字>` 每一块，都跑过一次带对应 `--envName` 的构建吗？
3. 新增覆盖块以后，在产物里 grep 过它那个值吗（`grep -rl "你的标记" .output`）？
4. 项目根有没有 `nitro.config.ts`、`vite.config.ts`、`postcss.config.js`、`webpack.config.ts`？有就搬进 `nuxt.config` 对应键，然后删掉旧文件。
5. 需要按环境换的值（密钥、地址）不该用 `$env` 解决——那是构建期；下一篇的 `runtimeConfig` 才是运行期那条通道。
6. 读当前环境名用 `import.meta.envName`，写在应用代码里；不要指望在 `nuxt.config.ts` 里读到它。

| 现象 | 先查什么 |
| --- | --- |
| 写了 `$staging: {…}` 一点反应没有 | 没有这个键。`$env` 下面才放自定义名字；而且 `$staging` 拼错也只会静默失效 |
| 覆盖块没生效，旗标也对 | 看是不是被更后写的同类键盖了；再确认你跑的那条命令带不带 `--envName` |
| `vite` 里的配置没进构建 | 项目根是否还留着一份 `vite.config.ts`；两份都在时只认 `nuxt.config` |
| 生成的 HTML 标题不是预期的那个 | 按顺序想：`.nuxtrc` → `nuxt.config` 根 → 当前 envName 选中的那块 |
| 线上想要另一个环境名 | 重新构建一次并带上那个 `--envName`，运行期设变量换不了 |

下一步 [731《工程基线（6）：环境变量进 runtimeConfig：用 zod 把 .env 变成有类型的东西》](/posts/915737a3a8c388133bf56e7c3ebbfe1d/)：把值这件事接到运行期——`NUXT_` 前缀怎么顶掉 `runtimeConfig` 里的默认值、哪些键客户端也读得到，以及用 zod 给这份配置加一道能跑的校验。

## 本文依据

- `.nuxtrc` 的扁平语法、`setups` 由 Nuxt 自动维护、`nuxt.config` 覆盖它、全局那份的位置与三级优先顺序：`https://nuxt.com/docs/4.x/directory-structure/nuxtrc`
- `nuxt.config` 的扩展名（`.js`/`.ts`/`.mjs`）、`defineNuxtConfig` 全局可用、改配置与 `.env`/`.nuxtignore`/`.nuxtrc` 会触发整站重启：`https://nuxt.com/docs/4.x/directory-structure/nuxt-config`
- 环境覆盖块 `$production`/`$development`/`$env`/`$meta`、`--envName` 示例、机制指向 `c12`：`https://nuxt.com/docs/4.x/getting-started/configuration`
- 外部配置文件两张表（四个划删除线、六个仍是独立文件）与 Vue 的四个配置位：`https://nuxt.com/docs/4.x/getting-started/configuration`
- `--envName` 的默认值那句话与五条命令的选项表：`https://nuxt.com/docs/4.x/api/commands/build`、`.../dev`、`.../preview`、`.../generate`、`.../prepare`
- `import.meta.envName`（v4.5，含自定义值）：`https://nuxt.com/docs/4.x/api/advanced/import-meta`
- `loadNuxt` 的 `envName: string | false` 与"优先于 `nuxt.config` 里的 envName"：`https://nuxt.com/docs/4.x/api/kit/programmatic`
- `envName` 这一位在配置里的完整来路（显式传入 → dev 取 `development`、否则取 `production` → 暴露为 `import.meta.envName`）出自本机装好的 `@nuxt/schema@4.5.2` 的类型注释，不是网页文档
- `app.baseURL` 默认 `/` 且可用 `NUXT_APP_BASE_URL` 运行期设置：`https://nuxt.com/docs/4.x/api/nuxt-config#baseurl`
- 本篇所有读数来自同一台机器上的最小工程（Node v24.11.1、npm 11.17.0、Nuxt 4.5.2），项目侧只读了 `Substrate/` 的两个配置文件，未作改动
