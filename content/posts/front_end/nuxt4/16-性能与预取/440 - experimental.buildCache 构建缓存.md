---
url: /posts/a61caf520628ab3cbb8f122ac022eedf/
title: 440 - experimental.buildCache 构建缓存
date: 2027-03-07T11:00:00+08:00
updated: 2027-03-07T11:00:00+08:00
author: cmdragon

summary:
  摘要：`experimental.buildCache` 按配置与源文件的哈希缓存 Nuxt 构建产物，默认关闭。它的边界官方写得很清楚：只对 `srcDir` 与 `serverDir` 里的 Vue/Nitro 源码生效；`srcDir` 内任何改动都会重建 Vue 客户端/服务端包；**Nitro 每次一定会重建**。有九个文件的变动会直接触发全量重建——`.nuxtrc`、`.npmrc`、`package.json` 与四种锁文件、`tsconfig.json`，以及 bun 的两份锁文件。缓存 tarball 最多保留 10 份。这一篇把这九项和"什么情况下开了也没用"讲清。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建缓存
  - experimental
  - 构建
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

CI 上每次构建都要等满几分钟，你搜到 `experimental.buildCache`，打开配置跑第二次——时间没变。

这不是配置写错了，而是这一项的作用范围被官方限定得很窄。开之前先看清它缓存什么、什么会打穿缓存。

## 它缓存的对象

官方第一句是定义：

> Caches Nuxt build artifacts based on a hash of the configuration and source files.

第二句是范围限定，这句最要紧：

> This only works for source files within `srcDir` and `serverDir` for the Vue/Nitro parts of your app.

也就是两个目录里的源码：`srcDir`（默认 `app/`，见 079 那篇的基准关系）与 `serverDir`。**默认关闭**，开法只有一种：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    buildCache: true,
  },
})
```

## 九个文件一动就全量重建

官方列出了"改这些文件会触发一次完整重建"的清单，一共九项：

```bash [Directory structure]
.nuxtrc
.npmrc
package.json
package-lock.json
yarn.lock
pnpm-lock.yaml
tsconfig.json
bun.lock
bun.lockb
```

这张表的读法值得说明：它不是"这些文件参与缓存计算"，而是**碰了就不复用**。`package.json` 和 `tsconfig.json` 在列，`nuxt.config.ts` 反倒不在——因为配置内容本身是哈希的一部分（定义那句写的是"a hash of the configuration and source files"）。

清单里六个是锁文件或 `.npmrc`：换依赖、换包管理器、动 registry 配置，缓存直接作废。`bun.lock` 与 `bun.lockb` 各占一行，是同一件事的两种格式（文本与二进制）。

## 命中缓存之后还剩多少活

官方的下一句把两种情况分开讲了：

> In addition, any changes to files within `srcDir` will trigger a rebuild of the Vue client/server bundle. Nitro will always be rebuilt (though work is in progress to allow Nitro to announce its cacheable artifacts and their hashes).

两条结论：

**`srcDir` 里任何改动都会重建 Vue 的客户端和服务端包。** 换句话说，你改一个页面文件，前端那一半必然重做；这项缓存省的不是这一轮。

**Nitro 永远重建。** 括号里那句"work is in progress"说明这是当前实现的限制，不是用法问题——服务端那部分产物现在还不会被声明为可缓存。

再补一条容量限制，官方用 `::note` 写着：**最多保留 10 份缓存 tarball**。

## 所以什么时候值得开

把上面三条合起来看，判断就具体了：

| 你的改动 | 缓存能省掉什么 |
| --- | --- |
| 只改 `srcDir` 里的源码 | 省不掉 Vue 包重建；Nitro 照样全重建 |
| 改那九个文件里的任何一个 | 全量重建，缓存直接作废 |
| 反复构建同一份未改动的代码 | 这是它设计上唯一覆盖的场景 |

也就是说，它对应的是"同一份代码构建第二次"，不是"改了代码构建变快"。**官方没有给任何加速比例或适用场景建议**，所以要不要开，只能用你自己的构建日志验证：连续两次 `nuxt build`，第二次耗时和第一次一样，就说明这一项在你的项目上拿不到收益。

顺带一提，`nuxt cleanup` 那条命令清的是构建目录、`.output`、`dist`、`node_modules/.vite` 与 `node_modules/.cache` 这五处，官方**没有把 buildCache 的存储路径写进这两页里**（文档未说明），所以别拿 `cleanup` 当作"清构建缓存"的确定手段——它清了哪些是明确的，是否清掉这项缓存不明确。

## 自查

- 定义是按配置与源文件哈希缓存产物；默认关闭；只对 `srcDir` 和 `serverDir` 的 Vue/Nitro 部分生效。
- 九个文件变动即全量重建：`.nuxtrc`、`.npmrc`、`package.json`、四种锁文件、`tsconfig.json`、`bun.lock`、`bun.lockb`。
- `srcDir` 改动一定重建 Vue 客户端/服务端包；Nitro 一定重建（实现限制，官方标注在做）。
- 最多留 10 份 tarball。
- 收益只出现在"同一份代码重复构建"，且没有官方加速数字，要用两次构建实测。

## 下一篇

下一篇 441 讲 `optimization` 这个配置节。它下面只有三个键，都不是性能开关：前两个决定编译期怎么处理异步上下文和给函数注入 key，第三个的默认值里能看到一批生命周期钩子被从构建中摇掉——那正是"服务端跑不到 `onMounted`"这类规则的实现出处。

## 本文依据

- `## buildCache` 全节：定义原文、"only works for source files within `srcDir` and `serverDir`"、默认关闭与启用示例、触发全量重建的九个文件清单原文、`srcDir` 改动重建 Vue 客户端/服务端包、"Nitro will always be rebuilt"及其 work-in-progress 括号说明、`::note` 最多 10 份 cache tarballs（`guide/going-further/experimental-features`）
- `nuxt cleanup` 清理的五处清单（`api/commands/cleanup`）
- `srcDir`、`serverDir` 与 `buildDir` 的基准关系（`api/nuxt-config`、`directory-structure` 相关页）
