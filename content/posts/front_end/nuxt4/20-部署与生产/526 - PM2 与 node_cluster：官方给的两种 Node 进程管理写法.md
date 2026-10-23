---
url: /posts/cd287ca145be62c556c2f27c9630e078/
title: 526 - PM2 与 node_cluster：官方给的两种 Node 进程管理写法
date: 2027-04-05T10:00:00+08:00
updated: 2027-04-05T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：让一个 Nuxt 产物跑多个进程，官方在部署页给的两条写法都只有一段：PM2 那份 `ecosystem.config.cjs`（六个字段：`name`、`port: '3000'`、`exec_mode: 'cluster'`、`instances: 'max'`、`script: './.output/server/index.mjs'`、`env.NODE_ENV: 'production'`），以及 `NITRO_PRESET=node_cluster`——官方对后者的说法是"用 Node.js 的 cluster 模块拿多进程性能"，并补了一句负载分配策略：**默认按 round robin 分给 worker**。两处都没解释：这两个 cluster 是不是同一件事、能不能同时用、PM2 的 `port` 字段与 `NITRO_PORT` 谁生效。

categories:
  - 前端开发

tags:
  - nuxt4
  - pm2
  - 进程管理
  - 部署
  - node
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一台机器上跑一个 Node 进程，多核就用不满。部署页为这件事给了两段写法，各自不到五行——然后就没了。

## 第一条：PM2

官方的定位句很短：**"PM2 (Process Manager 2) is a fast and easy solution for hosting your Nuxt application on your server or VM."**——适用面写的是你自己的服务器或虚拟机。

用法一句："To use `pm2`, use an `ecosystem.config.cjs`"，然后整份配置：

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

六个字段逐个看：

| 字段 | 值（官方示例） | 这一份配置里的作用 |
| --- | --- | --- |
| `name` | `'NuxtAppName'` | 应用名，进程列表里认它 |
| `port` | `'3000'` | **字符串**形式的端口，与部署页"默认监听 3000"一致 |
| `exec_mode` | `'cluster'` | 进程模式 |
| `instances` | `'max'` | 实例数量取最大 |
| `script` | `'./.output/server/index.mjs'` | **入口就是生产产物那个文件**，不是 `nuxt` 命令 |
| `env` | `{ NODE_ENV: 'production' }` | 把 `NODE_ENV` 写在配置里，而不是命令前缀 |

最后一行值得单独看：前面讲 `NODE_ENV` 时官方用的是命令前缀写法，而这份 PM2 配置把它放进 `env` 块——**同一个必需值，两条写入路径**，效果都是启动时它已经设好。

`script` 那行也与"产物自包含"对得上：`node_modules` 独立的那份 dist 直接当入口，PM2 不需要 `npm run` 任何脚本，因此也不依赖 `package.json` 里那五个脚本存在。

## 第二条：node_cluster

Cluster Mode 小节的原文两句：

> You can use `NITRO_PRESET=node_cluster` in order to leverage multi-process performance using Node.js cluster module.
>
> By default, the workload gets distributed to the workers with the round robin strategy.

两点信息：**这是 Nitro 的预设**（写法与 `node-server`、`vercel`、`netlify` 同族，走 `nuxt build` 时带环境变量），它的机制是 Node.js 自己的 cluster 模块；**负载分配默认 round robin**，即轮询分给各 worker。

对照前面 PM2 那份的 `instances: 'max'`：PM2 说的是"开多少个"，`node_cluster` 说的是"用哪种打包/运行预设"。

## 两处文档没回答的问题

这一段必须老实标出来，否则容易照抄出冲突配置：

**第一，两个 cluster 是不是同一件事？** PM2 的 `exec_mode: 'cluster'` 是进程管理器的概念，`NITRO_PRESET=node_cluster` 是 Nitro 预设的概念。**官方在两处都没有提到对方**，也没写"同时设置会怎样"或"该选哪个"。

**第二，端口字段谁生效？** 部署页的运行变量那节写的是 `NITRO_PORT` 或 `PORT`（默认 `3000`），而 PM2 示例里有一个 `port` 字段，值恰好也是 3000。**这两者的优先级文档没有写**——`port` 是不是被 PM2 转成 `PORT` 环境变量传下去，属于 PM2 的行为，不在这份文档范围内。要确认，最短的做法是只改其中一个、启动后看监听端口。

同理，`instances: 'max'` 与 cluster 预设各自开多少进程、会不会叠乘，官方也没写。

## 该配哪一条

按文档能给出的判断只有一条：**两条写法面向的都是"你自己的服务器或虚拟机"**。PM2 那条官方形容为 fast and easy；`node_cluster` 那条只说能拿到多进程性能，没提额外依赖。

至于要不要用进程管理器，官方没有给判断标准。可参照的事实是：部署页在讲 SSL 时明确建议把 Nitro 服务器放在**反向代理（nginx 或 Cloudflare）后面**、由代理终结 TLS——如果你的架构里已经有一层代理，端口与主机这两个默认值（`3000`、`0.0.0.0`）就是代理要对接的东西，这时进程管理只是本机事务。

## 自查

- PM2 那份是六字段配置，入口指向 `./.output/server/index.mjs`（因此不依赖 `package.json` 脚本），并把 `NODE_ENV: 'production'` 写进 `env` 块。
- `port: '3000'` 是字符串形式，与部署页的默认端口一致；`instances: 'max'` 决定进程数量。
- `NITRO_PRESET=node_cluster` 是 Nitro 预设，机制是 Node.js cluster 模块，负载默认 round robin。
- **两条路的关系官方没写**：两个 cluster 概念是否同一件事、能否叠加、`port` 与 `NITRO_PORT`/`PORT` 谁生效，都需实测，别当成等价配置。
- 两条都只面向自己的服务器或 VM；官方另一条建议是生产把服务放在反向代理后面、由代理终结 SSL。

## 下一篇

下一篇 527 讲那份"保留浏览器地址"的服务端插件：反向代理把同一个渲染结果暴露在多个公开路径时，水合阶段会发生什么、`delete nuxtApp.payload.path` 为什么能保住地址，以及官方给这条写法划的使用前提。
