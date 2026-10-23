---
url: /posts/b72a93f60fd6a77994590f5f8be4fa02/
title: 084 - watch 与 watchers：热重载不触发时先看这两个基准
date: 2026-11-09T09:00:00+08:00
updated: 2026-11-09T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方把热重载相关的配置分成两层。watch 定义"哪些模式变化时会重启 Nuxt 开发服务器"，它是一个字符串或正则的数组，而官方对基准的说明是：字符串应当是绝对路径，或者相对于 srcDir（以及任何图层的 srcDir）；正则则会针对相对于项目 srcDir（以及任何图层 srcDir）的路径来匹配——注意这一族的基准是 srcDir，与 079 那批相对 rootDir 的配置项不同。watchers 用来覆盖 watcher 自身的配置，它有三个子项：chokidar 是直传给 chokidar 的选项，官方给了两项默认值都是 true，即 ignoreInitial 与 ignorePermissionErrors；rewatchOnRawEvents 是一组事件类型，收到这些事件时会让 watcher 重启；webpack 是把 watchOptions 直传给 webpack，其下 aggregateTimeout 默认为 1000。配合 dev 命令页那条"reload 原地发生、restart 替换进程，而 nuxt.config 或已安装依赖的变化需要 restart"，以及 072 说的四个触发完全重启的文件，就能把"改了没反应"分成三类：文件不在监听基准内、事件被 watcher 选项吞掉、以及需要 restart 而不是 reload。

categories:
  - 前端开发

tags:
  - nuxt4
  - 热重载
  - chokidar
  - watch
  - 配置

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

我改了文件，页面没变。这句话在 Nuxt 里其实有三种不同的病，文件没被盯着、事件被吞了、需要的是重启而不是热替换。这一篇的 `watch` 与 `watchers` 分别治前两种，第三种在 033 已经讲过。

`watch`，往监听列表里加东西。官方原话，`watch` 属性让你定义那些模式发生变化时会重启 Nuxt 开发服务器的模式。它是一个字符串或正则的数组。字符串应当是绝对路径，或者相对于 `srcDir`，以及任何图层的 `srcDir`。正则则会针对相对于项目 `srcDir`，以及任何图层 `srcDir`，的路径进行匹配。

这一条最重要的信息是基准，`srcDir`。对照 079 那张表，`srcDir`、`serverDir`、`typesDir`、`modulesDir` 全是相对 `rootDir`，只有 `watch` 是相对 `srcDir`。

这就是一个真实的坑，默认 `srcDir` 是 `app`，051，所以你想盯 `server/` 或 `shared/` 里的东西，基准不是你以为的那一层，写 `watch: ['../some-dir']` 这种从 `app/` 往上退的路径会出现。这一句是我按基准是 `srcDir`、默认值是 `app` 这两个官方事实推出的写法后果。

还有一半信息在括号里，以及任何图层的 `srcDir`，068 讲过图层可以有与标准应用相同的结构，这里官方明确，图层的 `srcDir` 也在同一套基准之内。

`watchers`，三个子项，各治一类没反应。官方原话，`watchers` 属性让你覆盖 watchers 的配置在你的 `nuxt.config` 里。

一，`watchers.chokidar`。官方原话，直接传给 `chokidar` 的选项。

官方在这一节列了两项，默认值都是 `true`。

| 选项 | 默认 | 含义，照官方给的键名 |
| --- | --- | --- |
| `ignoreInitial` | `true` | 初始扫描时不逐个触发事件 |
| `ignorePermissionErrors` | `true` | 忽略权限错误 |

`ignorePermissionErrors: true` 这一条值得展开一句风险提示，官方在这里只给键名与默认值，下面这句是我按默认值方向给的使用提醒，它默认会安静地吞掉权限错误。而在两类场景里，权限错误恰恰是文件根本没法被监视的原因，容器或挂载卷，032 那份 `mounts` 把 `node_modules` 挂成卷，以及跨文件系统。

于是这条排查动作成立，热重载莫名不工作、且你不在裸文件系统上时，把 `ignorePermissionErrors` 设成 `false` 试一次，让错误显示出来而不是被忽略。

二，`watchers.rewatchOnRawEvents`。官方原话，一个事件类型数组，当收到这些事件时，会让 watcher 重启。

这条是给某些文件系统在 raw 事件模式下才发得出来 的情况准备的，官方在这里只给这一句定义，没列事件类型清单，所以我照原样给你，不补取值。

三，`watchers.webpack`。官方原话，直接传给 webpack 的 `watchOptions`。`aggregateTimeout`，类型 `number`，默认 `1000`。

这一项有前提，它属于 webpack builder，088 讲构建器可切换、默认是 Vite。默认那 1000 毫秒就是把短时间内的多次改动聚合成一次重建。

换句话说，你留在 Vite 上，默认，这一项不影响你，这一句是官方直传给 webpack 的字面含义，不是新规则。

第三类没反应，需要的是 restart，不是 reload。033 抄过、072 又用了一次的那段。

当 dev 服务器 reload 或 restart 时，它会说明是什么引起的，而 `nuxt.config` 的变化还会列出真正不同的那些键。reload 原地发生，restart 替换进程，这是 `nuxt.config` 或已安装依赖的变化所需要的。

加上 072 那四个触发完全重启的文件，`nuxt.config`、`.env`、`.nuxtignore`、`.nuxtrc`，判断链条就完整了。

一张分诊表，改文件没反应时按顺序查。分诊顺序是我的组织，每一格的内容都是官方原文。

| 症状 | 先查什么 | 官方依据 |
| --- | --- | --- |
| 终端里没有任何因为什么重启的说明 | 该文件是否在监听范围内，需要 `watch` 加一条 | `watch` 的定义 |
| 加进 `watch` 了但仍不生效 | 基准写错了，`watch` 相对 `srcDir`，不是 `rootDir` | 那句相对于 `srcDir` 以及任何图层的 `srcDir` |
| 容器或挂载卷里莫名其妙不监视 | `watchers.chokidar.ignorePermissionErrors`，默认 `true` 会忽略权限错误 | 该项默认值 |
| 初始那批事件重复触发 | `watchers.chokidar.ignoreInitial`，默认 `true` | 该项默认值 |
| 改 `nuxt.config`、装依赖之后要新进程 | 这是 restart 的活，不是热替换 | reload 原地、restart 替换进程 |
| 只想验证一次到底有没有被重编译 | 033 的 `shift-r`，清空缓存后重启，必要时 040 `cleanup` 加 035 `prepare` | 那两页 |
| 用 webpack 时批量改动只重建一次 | `watchers.webpack.aggregateTimeout`，默认 1000ms | 该项定义 |

顺手一条，Windows 的慢不是这里的配置问题。023 抄过官方那两条 Windows 建议，HMR 很慢可以试 WSL，官方措辞是可能会解决一些性能问题，以及地址栏用 `127.0.0.1` 代替 `localhost:3000`，官方标注的是浏览器侧的慢 DNS 解析。

慢不等于不触发。这两件事别混，没反应查这一篇，很慢查那两条。

## 自查

- [ ] `watch` 数组里一个字符串，基准是什么？正则呢？
- [ ] 为什么 `watch` 与 `serverDir` 的基准不一样？这对写路径意味着什么？
- [ ] 图层里的文件与这个基准是什么关系？官方哪半句写了它？
- [ ] 哪个 watcher 选项默认会吞掉权限错误？默认值是多少？
- [ ] `aggregateTimeout` 在哪种构建器下才有意义？默认值是多少毫秒？
- [ ] 改了 `nuxt.config` 之后期望的是 reload 还是 restart？依据是哪句话？

## 下一篇

085 讲 `devServer`：端口与主机的默认值、`url` 为什么官方写着不要直接设置它、`https` 在这份配置里怎么写，以及 `devServerHandlers` 那类只在开发期存在的接口。
