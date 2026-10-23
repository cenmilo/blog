---
url: /posts/ecacebdf1aa928b1b4c99d6ae0e66467/
title: 044 - nuxt upgrade：改版本、重装依赖、清构建缓存一条命令做完
date: 2026-10-26T11:00:00+08:00
updated: 2026-10-26T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方对 nuxt upgrade 的定义把它的动作列了三件——它更新 package.json 里的版本、用你的包管理器重新安装依赖，并移除构建缓存；这条命令不必手动配合 030 里"删锁文件重装"那套动作。选目标版本靠 --channel，它的默认值是 stable（最新发行版），可选 v3 或 v4 停在某个大版本，还有 nightly、v3-nightly、v4-nightly 三个夜间频道；--force 会从零重建锁文件与 node_modules，--dedupe 则在包管理器支持的情况下于升级之后去重依赖。最容易被漏掉的是 pnpm catalog 那条：一个被钉在 pnpm catalog 里的依赖（形如 "nuxt": "catalog:"）会在 pnpm-workspace.yaml 里升级——版本号实际住在那儿，而不是在 package.json。

categories:
  - 前端开发

tags:
  - nuxt4
  - 升级
  - 版本
  - pnpm
  - 命令行

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

016 讲版本规则时我说过什么时候必须升，030 讲锁文件时我说过切频道要删锁文件重装。这一篇是把那两件事合并成一条命令，`nuxt upgrade`。

官方定义，它做三件事。`upgrade` 命令把 Nuxt 升级到最新版本。它会更新你 `package.json` 里的版本、用你的包管理器重新安装依赖，并移除构建缓存。

这三件里第二件最容易被忽略，它不是只改一下版本号。它会跑你的包管理器重装，030 那张锁文件对照表所描述的那整套动作，由这条命令一起做掉。

所以这条命令的副作用范围要说清楚，它会动 `package.json`、`node_modules` 和锁文件。这是它比前面几条命令都要重的原因，下面第 4 节我会给出我建议的动作顺序。

`--channel`，升级目标的六种选法。官方说明是，`--channel` 决定升到哪儿，`stable` 是最新发行版，`v3` 或 `v4` 让你停在某个大版本上，`nightly`、`v3-nightly` 或 `v4-nightly` 走夜间发布频道。

选项表里给默认值，`-ch, --channel` 默认 `stable`，指定从哪个频道安装。

这张表里有两个实用结论。

一，默认就是 `stable`。你什么参数都不加，它把你送到最新发行版，而不是最新 nightly。这一点很重要，因为 016 里那条警告说的是另一回事，nightly 频道很可能有破坏性变更。默认值把你挡在这个风险之外。

二，停在当前大版本是有专门通道的。`v3`、`v4` 这两个值就是为这个需求准备的，想升小版本不想跨大版本，不必手写版本号，用 `--channel=v4`。

对照 016 讲的 nightly 手工做法，那一篇里要自己改 `package.json` 成 `"nuxt": "npm:nuxt-nightly@latest"`，然后删锁文件、重装。现在看这条命令，`--channel=nightly` 走的是同一件事，而且按官方定义它把重装依赖加清构建缓存也一起做了。手工那套是三处都要动，命令这一条覆盖全部。

`--force` 与 `--dedupe`。官方的两句是，`--force` 从零重建锁文件与 `node_modules`，而 `--dedupe` 在包管理器支持的情况下，于升级之后对依赖去重。

`-f, --force` 是强制升级，重新创建 lockfile 与 node_modules。`--dedupe` 是升级后对依赖去重。

两条的适用时机很不一样。

`--force` 是锁文件本身可疑时用。因为它会删掉锁文件重建，这意味着依赖的精确版本会重算，这是解决我这装出来跟 CI 不一样最狠的一招，但也是最不该随手用的一招。

`--dedupe` 的措辞里有个限定，where the package manager supports it，在包管理器支持的范围内。也就是说它不是所有管理器都有效，官方没有在这里列出哪几个支持，这一点与 030 那张五种包管理器表连起来读，不同管理器的能力差别是官方明确存在的。跑完不要假设依赖一定被去重了。

pnpm catalog，版本不住在 `package.json`。这一句是整篇里最容易白跑一趟的地方，官方单独给了一段，一个被钉在 pnpm catalog 里的依赖，`"nuxt": "catalog:"`，会在 `pnpm-workspace.yaml` 里升级，版本号实际住在那里，而不是在 `package.json` 里。

实操含义有三层。

如果你的 `package.json` 里 `nuxt` 的值是 `catalog:`，那你在 `package.json` 里看不到版本，去 `pnpm-workspace.yaml` 找。

官方这条描述意味着 `nuxt upgrade` 已经处理了这种情况。你不必手工改两处，但值得跑完确认一下改的是不是你预期那个文件。

这个文件名你已经见过两次了，030 的 devcontainer Dockerfile 里 `COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./` 就把它列进拷贝清单，pnpm workspace 项目里它是一等公民。

`catalog:` 这个语法本身是 pnpm 的特性，官方在这里只说版本号实际住在那里，它的全部规则不在 Nuxt 文档里，我不替 pnpm 展开。

建议的动作顺序，这是我的，不是文档给的。把 016、030、035、040 串起来，我会这样跑。

```bash [Terminal]
npx nuxt info
```

先收下升级前的基线，042 讲过，版本、包管理器、模块清单。

```bash [Terminal]
npx nuxt upgrade
```

按默认 `stable` 升级。

```bash [Terminal]
npx nuxt prepare
```

重装依赖后让类型重新生成，035 讲过，正常流程由 `postinstall` 兜住，但升级路径值得显式跑一次确认。

```bash [Terminal]
npx nuxt typecheck
```

这一步才是升级到底有没有出事的第一道真判据，035 讲过 `dev` 与 `build` 默认都不检查类型。

```bash [Terminal]
npx nuxt info
```

再收一次，和基线对比。

为什么顺序不能反，`upgrade` 会重装依赖并清构建缓存，这一步之后 `.nuxt` 里的类型可能已经不存在了，对照 040 那张清单里的构建目录一项。跳过 `prepare` 直接 `typecheck`，就是你亲手制造 036 的第二查那个坑。

## 自查

- [ ] 官方说 `nuxt upgrade` 做哪三件事？它会不会动 `node_modules`？
- [ ] 想升到最新**小版本**而不跨大版本，用哪个值？不加 `--channel` 时默认升到哪儿？
- [ ] `--force` 会重建什么？为什么不该随手用？
- [ ] `--dedupe` 的措辞里有一个限定条件，是什么？
- [ ] 你的 `package.json` 里 `nuxt` 写着 `catalog:`，那该去哪个文件改版本？

## 下一篇

045 讲两条「装东西」的命令：`nuxt add` 与 `nuxt module`——模块、图层分别怎么装、怎么搜、怎么卸载。
