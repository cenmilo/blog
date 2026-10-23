---
url: /posts/9f8a3c750e1412828064c85d6dfb9d36/
title: 523 - 官方那份 package.json 里的五个脚本与启动方式
date: 2027-04-04T10:00:00+08:00
updated: 2027-04-04T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方 `directory-structure/package` 那一节给的最小 `package.json` 里有五个脚本：`build: nuxt build`、`dev: nuxt dev`、`generate: nuxt generate`、`preview: nuxt preview`、**`postinstall: nuxt prepare`**，依赖只有 `nuxt`、`vue`、`vue-router` 三项，另有 `private: true` 与 `type: "module"`。最后那个脚本最要紧：`nuxt prepare` 创建 `.nuxt` 目录并生成类型，官方明说它适合 CI 或作为 `postinstall`。生产启动不走这些脚本——PM2 那份配置里 `script` 字段直接指向 `./.output/server/index.mjs`。要提醒一句：**整套 4.x 文档里没有任何"健康检查端点"的记载**，最接近的东西是服务端路由那套 `/api` 前缀规则，你得自己写那个文件。

categories:
  - 前端开发

tags:
  - nuxt4
  - package
  - 启动脚本
  - 部署
  - 健康检查
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

平台上配启动命令时，很多人把 `npm run build && npm start` 抄进流水线，然后发现根本没有 `start` 这个脚本。原因就在官方那份最小 `package.json` 里——它给的脚本只有五个。

## 官方那份最小 package.json，逐行

那一节的引导句是 **"The minimal `package.json` of your Nuxt application should looks like"**（原文如此），内容整份如下：

```json [package.json]
{
  "name": "nuxt-app",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "nuxt build",
    "dev": "nuxt dev",
    "generate": "nuxt generate",
    "preview": "nuxt preview",
    "postinstall": "nuxt prepare"
  },
  "dependencies": {
    "nuxt": "latest",
    "vue": "latest",
    "vue-router": "latest"
  }
}
```

五个脚本与四条命令页能对上，各自的作用：

| 脚本 | 跑的是什么 | 命令页给的定性 |
| --- | --- | --- |
| `dev` | `nuxt dev` | 开发服务器（含热更新） |
| `build` | `nuxt build` | 创建 `.output`，把应用、服务器与依赖准备好生产，并报告耗时 |
| `generate` | `nuxt generate` | 预渲染每条路由，结果存成纯 HTML 文件，可部署到任意静态托管 |
| `preview` | `nuxt preview` | 在 build 之后起一个服务器来预览应用 |
| `postinstall` | `nuxt prepare` | 创建 `.nuxt` 目录并生成类型 |

**没有 `start`。** 需要"起服务"时官方用的是 `nuxt preview`（并说明 `nuxt start` 是同一条命令的另一个名字），而真正的生产启动走的是部署页那条 `node .output/server/index.mjs`。

## postinstall 那一行为什么重要

`prepare` 命令页那句解释正好落在这个脚本上：

> The `prepare` command creates a `.nuxt` directory in your application and generates types. This can be useful in a CI environment or as a `postinstall` command in your `package.json`.

也就是说这一行不是装饰：装完依赖就生成 `.nuxt` 与类型，编辑器与 `vue-tsc` 才认 `~/`、`#imports`、`#components`。流水线里如果没有 `npm install` 这一步（只复用缓存好的产物目录），这一行不会触发，类型检查就会失败——这是官方把它写成默认脚本的理由。

另外两个字段官方只写不给解释：`private: true` 与 `type: "module"`。**这一节没有说明它们的作用**，所以本系列不替文档补语义。能确证的只有：`type: "module"` 与 ESM 那一节对产物的描述在同一套语境里——Nitro 输出 `.output/server/index.mjs`，官方说"那会让 Node.js 把这个文件当原生 ES 模块处理"。

依赖那三项的版本写法官方示例里就是 `"latest"`，不是区间或精确版本号——这一节没讨论锁版本策略。

## 生产启动不读这些脚本

部署页给的启动形状有两种，都不经过 `package.json`：

```bash [Terminal]
NODE_ENV=production node .output/server/index.mjs
```

以及 PM2 那份 `ecosystem.config.cjs`，其中 `script` 字段就是同一个文件：

```ts [ecosystem.config.cjs]
module.exports = {
  apps: [
    {
      name: 'NuxtAppName',
      port: '3000',
      exec_mode: 'cluster',
      instances: 'max',
      script: './.output/server/index.mjs',
      env: {
        NODE_ENV: 'production',
      },
    },
  ],
}
```

读这张配置能得到四件部署事实：进程管理器直接把 `.mjs` 当入口、端口在 `env` 之外还能显式给、`exec_mode` 与 `instances` 是官方式样（cluster／max）、**`NODE_ENV: 'production'` 在这里是写在 `env` 块里的**——与部署页那条 `::important` 一致：这个值得你自己带，不设会被 Vue Router 的开发期警告灌日志。

## 健康检查：文档没有这一项

先把没有的说清楚，避免照着别处经验配：**整套 4.x 文档里没有 health check、readiness、liveness 这一类端点的记载**，没有内置状态路由，也没有任何"平台探测应该指向哪个地址"的建议。

文档里最接近"状态端点"的形状，出现在模块文档的示例中：一个服务端处理函数返回一个对象（例如把模块自己的版本读出来）。要给自己做一个，能依的只有目录规则那两条：**`server/api/**` 下的文件自动带 `/api` 前缀，`server/routes/**` 不带**。

```ts [server/api/health.get.ts]
export default defineEventHandler(() => ({ status: 'ok' }))
```

**上面这段是本系列按上述规则给的写法示例，官方没有给健康检查示例。** 三点提醒：文件后缀 `.get` 属于 Nitro 的"按 HTTP 方法匹配路由"能力，不是 Nuxt 在这一节讲的；这个端点会走服务端渲染那条路径，纯静态部署（`nuxt generate`）下它不存在；如果给它加了缓存类路由规则，探测结果就不再反映进程状态。

平台侧要配探测地址时，你能确证的只有"这个路径由你自己定义"，官方没有默认值可指。

## 自查

- 官方最小 `package.json` 只有五个脚本：`dev`/`build`/`generate`/`preview`/`postinstall: nuxt prepare`，**没有 `start`**。
- `postinstall` 那行的作用是生成 `.nuxt` 与类型，官方明说 `prepare` 适合 CI 或作为 `postinstall`；跳过安装的流水线要自己补这一步。
- `private` 与 `type: "module"` 的语义这一节没解释，别按推测写成结论。
- 生产启动绕过脚本：`node .output/server/index.mjs`，或 PM2 的 `script` 指向同一个文件，`NODE_ENV` 在 `env` 块里显式给。
- **健康检查端点没有任何官方记载**；要做就按 `server/api` 的 `/api` 前缀规则自己写一个，并注意静态部署下它不存在。

## 下一篇

下一篇 524 讲上线前该核对的四件事：错误 `cause` 在生产不会外泄、chunk 加载失败有内置处理、启动错误与 Nitro 服务端错误各自能钩到哪一层。
