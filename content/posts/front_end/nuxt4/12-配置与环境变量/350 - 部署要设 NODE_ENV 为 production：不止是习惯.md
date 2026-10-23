---
url: /posts/25e89446e139541f48788c955c8fc801/
title: 350 - 部署要设 NODE_ENV 为 production：不止是习惯
date: 2027-02-05T11:00:00+08:00
updated: 2027-02-05T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：用 Node 服务器 preset 部署时，启动命令要带 NODE_ENV=production，这不是可选的仪式感。官方 deployment 页给了具体后果：一些依赖（尤其点名 Vue Router）只有在这个变量设了之后才会剥掉开发期专用的告警，不设就会在遇到匹配不到的路由时，往日志里刷一堆 "[Vue Router warn]: No match found for location with path …"。这一篇讲启动命令的正确形状、它顺带尊重哪些运行时环境变量（端口 NITRO_PORT/PORT、host NITRO_HOST/HOST、SSL 证书），以及 PM2/cluster 下怎么带这个 env。

categories:
  - 前端开发

tags:
  - nuxt4
  - 部署
  - NODE_ENV
  - Vue Router
  - Node preset
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`nuxt build` 出来一个 `.output`（331）。用 Node 服务器方式跑它时，官方给的启动命令是这样：

```bash [Terminal]
NODE_ENV=production node .output/server/index.mjs
```

`NODE_ENV=production` 这一段，很多教程一带而过，其实它有具体后果。

## 为什么必须带它

deployment 页的 important 说得很直接：

> Set `NODE_ENV=production` when running the server. Some dependencies (notably Vue Router) only strip development-only warnings when this is set, so leaving it unset can flood your logs with messages like `[Vue Router warn]: No match found for location with path …` on unmatched routes.

拆开看：

- 一些依赖（官方点名 Vue Router）里有"只在开发模式出现的告警"，靠 NODE_ENV 判断要不要保留。
- 你不设 `production`，这些开发期告警不会剥掉。
- 后果很具体：一旦有请求打到"匹配不到任何路由"的路径，日志就被 `[Vue Router warn]: No match found ...` 刷满。

也就是说漏设它不是"少个标记"，而是**生产日志被开发告警灌爆**。337 那种带鉴权重定向、有爬虫乱撞路径的线上场景，最容易触发。

## 顺带：Node preset 认哪些运行时环境变量

官方列了这个入口点尊重的变量：

- `NITRO_PORT` 或 `PORT`（默认 `3000`）——监听端口。
- `NITRO_HOST` 或 `HOST`（默认 `'0.0.0.0'`）——监听地址。
- `NITRO_SSL_CERT` 和 `NITRO_SSL_KEY`——两者都在就启 HTTPS。但官方建议这只用于测试，生产应把它放在 nginx/Cloudflare 这类反向代理后面终止 SSL。

所以设端口/主机也是在这些运行时环境变量里设，别去改代码。

## PM2 与 cluster 里怎么带

如果用 PM2 托管，官方 `ecosystem.config.cjs` 示例把 NODE_ENV 写进了 env 块：

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

关键：不管你是命令行直接起、还是 PM2/cluster 起，**`NODE_ENV: 'production'` 都得设**。PM2 就放在它的 `env` 里。

想用 Node 原生 cluster 多进程，则是 `NITRO_PRESET=node_cluster`（官方："leverage multi-process performance"、默认 round robin 分发）——那是选 preset 的事，和 NODE_ENV 是两码。

## 一句提醒：别写成别的值

NODE_ENV 生效的判据是值等于 `production`。写成别的（或不设）都会让上面那些开发告警留着。这是"部署时最容易漏的一行环境变量"。

## 自查

- 用 Node preset 起服务，命令里带 `NODE_ENV=production` 了吗？不带会被 Vue Router 的 No match 告警刷日志。
- 用 PM2 的话，NODE_ENV 写进 ecosystem.config 的 env 了吗？
- 端口/主机用 `NITRO_PORT`/`PORT`、`NITRO_HOST`/`HOST` 设，别硬编进代码。
- 生产别直接用 NITRO_SSL_* 起 HTTPS，交给反向代理终止 SSL。

## 下一篇

下一篇 351 讲 compatibilityDate：Nuxt 靠它决定用哪批行为默认值，每个项目都该显式写，否则跨版本升级会悄悄换默认（对应错误码 B5001）。
