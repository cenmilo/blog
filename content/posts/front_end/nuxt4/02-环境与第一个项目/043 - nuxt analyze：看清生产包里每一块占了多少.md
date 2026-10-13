---
url: /posts/406a489d9ece74f5c4867d22a061b45b/
title: 043 - nuxt analyze：看清生产包里每一块占了多少
date: 2026-10-26T10:00:00+08:00
updated: 2026-10-26T10:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt analyze 会先构建 Nuxt 再分析生产包，官方把这条命令整体标注为实验特性；构建结束后结果默认由一个本地服务器提供，除非你传 --no-serve，或者环境里设了 CI。它一共七个开关，其中三个有明确的默认值：--serve 默认为 true、--prerender 默认为 false、--name 默认为 default，同样把 process.env.NODE_ENV 设为 production。关于 --prerender 官方给了一条容易误读的解释："分析期间不会预渲染路由，因为预渲染会运行构建好的应用，而它的输出并不是我们要测量的东西"——也就是说默认不预渲染不是遗漏，而是刻意：你要测的是包体，不是页面跑起来的结果，除非你的分析确实需要预渲染再显式加上。它和 037 的 --profile 是两件不同的事：--profile 量构建过程花了多久，analyze 量产物里每一块占了多少。

categories:
  - 前端开发

tags:
  - nuxt4
  - 包体分析
  - 性能
  - 命令行
  - 实验特性

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


037 结尾我留了一句区分：`--profile` 量的是**构建过程**花了多久，`nuxt analyze` 量的是**产物**里每一块占了多少。这一篇把后者讲完——它很短，但官方在正文里塞了一条很容易误读的解释，那条才是本篇的重点。

## 官方定义里有两个限定词

> `analyze` 命令会构建 Nuxt 并分析生产包（**实验特性**）。构建结束时，结果会在一个本地服务器上提供，**除非你传 `--no-serve`，或者环境里设置了 `CI`**。

拆开看：

1. **它会先构建。** 所以它不是"读一下现有 `.output`"——它跑一次构建再分析（对照 037：`.output` 是构建产物）。
2. **标注为 experimental。** 016 讲过官方对实验特性的承诺方式：**破坏性变更只会发生在明确标记为 experimental 的选项之内**。所以这条命令的输出形态、URL、参数以后都可能变；把它接进 CI 之前要清楚这一点。
3. **"默认起一个本地服务器"这件事有两个关闭条件**：显式 `--no-serve`，或者环境里有 `CI`。

第 3 条很实际：**同一条命令在你机器上开服务器、在 CI 上不开**——这不是 bug，是官方写明的默认行为。

## 全部选项

```bash [Terminal]
npx nuxt analyze [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--dotenv=<path>...] [-e, --extends=<layer-name>...] [--name=<name>] [--serve] [--prerender]
```

| 选项 | 默认 | 官方说明 |
| --- | --- | --- |
| `ROOTDIR` | `.` | 项目根目录 |
| `--cwd=<directory>` | | 指定项目根目录 |
| `--logLevel=<silent\|info\|verbose>` | | 构建期日志级别 |
| `--dotenv=<path>...` | | 要加载的 `.env`，可重复，**靠后的优先** |
| `-e, --extends=<layer-name>...` | | 从某个图层继承 |
| `--name=<name>` | `default` | 这次分析的名字 |
| `--serve` / `--no-serve` | `true` | 提供分析结果 / 跳过提供 |
| `--prerender` | `false` | 分析期间预渲染路由 |

末尾一条 note 与 build 一致：

> 本命令把 `process.env.NODE_ENV` 设为 `production`。

`--name` 是这八个里唯一容易忽略却对**比对**有用的：它的默认值是 `default`，也就是说**你可以给每次分析起名**（例如 `before-split`、`after-split`）。这一项官方只写了"分析的名字"，没说明名字会出现在哪里——**我不替它补充含义**，但从"有名字 vs 全是 default"这个事实出发，多次分析时给它一个有意义的名字是零成本的。

## 那条容易误读的解释：为什么默认不预渲染

官方专门写了一段，值得逐字看：

> 分析期间**不会预渲染路由**，因为预渲染会运行构建好的应用，而**它的输出并不是我们要测量的东西**。如果你的分析确实需要它，请传 `--prerender`。

这句话解决了两个具体误解。

**误解一："analyze 结果里没有 HTML，是不是坏了？"**
不是。默认它就不预渲染——按官方给的理由，它关心的是**包**，而预渲染的产物是"跑起来的应用的输出"，不属于被测量的对象。

**误解二："既然要生产包，那顺带预渲染不是更真实？"**
官方的答案恰好相反：预渲染会**多跑一遍应用**，而这一遍的结果不是测量目标。所以默认关是省时间且更聚焦。

**那什么时候该加 `--prerender`？** 官方把决定权给了你——"如果你的分析需要它"。**典型的需要是：你想知道预渲染阶段引入的依赖是否算进包体。**（这一句是我对"什么时候需要"的推测性举例；官方只说了"你的分析需要它"，没列举场景。）

## 和 037、044 的分工

三条命令的差别，用一张表收干净：

| 命令 | 量什么 / 干什么 | 官方标注 |
| --- | --- | --- |
| `nuxt build --profile` | 构建过程的时间去向（`nuxt-build.cpuprofile`、`perf-report.json`、`perf-trace.json`） | 完整报告需 Nuxt v4.4+ |
| `nuxt analyze` | 生产包里各部分占多少 | **实验特性** |
| `nuxt upgrade`（044） | 改版本、重装依赖、清构建缓存 | 会动 `node_modules` 与锁文件 |

**先知道"包为什么大"要看 analyze；先知道"构建为什么慢"要看 `--profile`。** 这条区分是这一篇存在的意义——两者都叫"分析类命令"，但输出的对象完全不同。

## 自查

- [ ] `nuxt analyze` 会不会自己先构建一次？它的官方标注是什么级别？
- [ ] 在你的终端里它会起一个本地服务器，在 CI 里不会——这是谁决定的？
- [ ] 为什么默认不预渲染？官方的理由是什么？
- [ ] 你连着跑两次分析，靠哪个参数把两次区分开？它的默认值是什么？
- [ ] 想知道"构建慢在哪"，该用这条命令还是 `build --profile`？

## 下一篇

044 讲 `nuxt upgrade`——官方说它一次做三件事：改 `package.json` 里的版本、用你的包管理器重装依赖、清掉构建缓存。还有一条 pnpm catalog 的特殊处理。

## 本文依据

- `api/commands/analyze`（analyze 构建 Nuxt 并分析生产包且标注 experimental、构建结束时结果在本地服务器提供除非传 --no-serve 或环境设置了 CI；"Routes are not prerendered while analyzing, since prerendering runs the built app and its output is not what is being measured. Pass --prerender if your analysis needs it."；用法行与选项 ROOTDIR 默认 .、--cwd、--logLevel、--dotenv 可重复靠后优先、-e extends、--name 默认 default（Name of the analysis）、--serve 默认 true 与 --no-serve、--prerender 默认 false；note 本命令设 process.env.NODE_ENV 为 production）
- `api/commands/build`（--profile 写 nuxt-build.cpuprofile，构建耗时与 perf-report.json/perf-trace.json 需 Nuxt v4.4 或更新；build 创建 .output 并为生产准备好——用于与 analyze 分工）
- `community/roadmap`（破坏性变更只发生在明确标记为 experimental 的选项内——用于解释 analyze 的 experimental 标注，见 016）
