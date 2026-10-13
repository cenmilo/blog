---
url: /posts/1d845bdfb066cc88cf7b291cd04f3a0e/
title: 495 - nuxt test 命令与它的三个选项
date: 2027-03-26T09:00:00+08:00
updated: 2027-03-26T09:00:00+08:00
author: cmdragon

summary:
  摘要：`npx nuxt test` 是 CLI 给出的测试入口，一句话定位是"用 `@nuxt/test-utils` 跑测试"，并且会在 `process.env.NODE_ENV` 尚未设置时把它设为 `test`。它只有一个位置参数 `ROOTDIR`（默认 `.`）和三个选项：`--cwd=<directory>`、`--dev`、`--watch`。需要注意的是 `ROOTDIR` 与 `--cwd` 的官方描述文字是一样的，文档没有说明同时给出时谁生效；而文档在测试这一侧主推的是 `npx vitest --project unit` 这类直接调运行器的写法，两条入口各自怎么配合，官方未记载。

categories:
  - 前端开发

tags:
  - nuxt4
  - cli
  - 测试
  - vitest
  - 命令行
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

配好 vitest 之后，跑测试你打的是 `npx vitest`。但 CLI 里还有一条 `nuxt test`，脚本里写哪一条、它能带什么，是两回事。

这一节很短，正好可以一次说完。

## 命令形状

```bash
npx nuxt test [ROOTDIR] [--cwd=<directory>] [--dev] [--watch]
```

官方对它的定义只有一句：**"The `test` command runs tests using `@nuxt/test-utils`."**

也就是说它不负责选运行器、不负责告诉你测什么，它是把 `@nuxt/test-utils` 那套东西接上项目、然后交给测试去跑。这条页里没有出现 vitest 字样，也没有 `--project` 之类的参数。

## 参数与选项全表

| 名称 | 写法 | 官方说明 | 默认 |
| --- | --- | --- | --- |
| `ROOTDIR` | 位置参数 | The root directory of your Nuxt project | `.` |
| `--cwd=<directory>` | 选项 | Specify the root directory of your Nuxt project | 表中留空 |
| `--dev` | 开关 | Run in dev mode | 表中留空 |
| `--watch` | 开关 | Watch mode | 表中留空 |

四行里没有第五行。这一节没列全局选项（`--log-level` 之类在这一节的表里不出现），要确认得用 `npx nuxt test --help` 在你自己的机器上看。

有一处描述重复值得留意：`ROOTDIR` 和 `--cwd` 的官方说明文字是同一句"你的 Nuxt 项目根目录"。**同时给出哪一个、以谁为准，文档未说明。** 实际项目里只写一个就好，别指望顺序规则。

## 它对环境变量做的那件事

这句话在这一节出现了两次——正文里一次，结尾的 `::note` 里又一次：

> This command sets `process.env.NODE_ENV` to `test` if not already set.

关键是后半句 "if not already set"。它是**只在没设过的时候才设**，不是强制覆盖。所以如果你在外部先 `NODE_ENV=production npx nuxt test`，这条命令不会替你改回来；你的代码里读 `process.env.NODE_ENV` 的那些分支（生产构建、`import.meta.env` 相关判断）就跟着你设的那个值走。

这也解释了为什么 CI 里常见的坑是"本地绿、线上红"的反面：你在脚本里显式设过 `NODE_ENV`，本地跑 `npx vitest` 时它没被设成 `test`，而换 `npx nuxt test` 之后行为变了。要复现同一种环境，两条入口得挑一条固定下来。

## 与直接跑 vitest 的关系

官方测试文档给出来的运行方式是直接调运行器，四条命令分别是：

```bash
# Run all tests
npx vitest

# Run only unit tests
npx vitest --project unit

# Run only Nuxt tests
npx vitest --project nuxt

# Run tests in watch mode
npx vitest --watch
```

对照上一条命令那一节，能确认的只有这些：`--watch` 这件事两边都能表达（`nuxt test --watch` 与 `npx vitest --watch`）；按测试种类分别跑，只有 `npx vitest --project <name>` 这一种写法有记载，`nuxt test` 那一节里没有对应选项。

**`nuxt test` 与 `vitest.config.ts` 里三个 project 之间具体怎么协作，文档没有记载**——它只说这条命令"用 `@nuxt/test-utils` 跑测试"。要判断它是否读你那份 project 配置，唯一的办法是自己跑一次看输出：`npx nuxt test --dev` 与 `npx vitest --project nuxt` 分别跑同一批测试，对比被执行的文件数。

## 自查

- `nuxt test` 的定位一句话：跑测试用的是 `@nuxt/test-utils`；这一节不提 vitest，也没有 `--project`。
- 只有 1 个位置参数加 3 个选项，全表四行；`--cwd` 与 `ROOTDIR` 描述相同，优先级文档未说明。
- `NODE_ENV` 是「未设置时才设为 `test`」，外部已经设过就不覆盖——这是 `nuxt test` 与裸 `npx vitest` 最容易产生的差异。
- 分项目跑测试只在 `npx vitest --project unit` 这类写法里有记载。
- 全局选项这一节没列，用 `--help` 在本地确认，别照其它命令那一节类推。

## 下一篇

下一篇 496 讲 `nuxt.config.ts` 里的 `test` 这一节：它的类型、默认值，以及它管的事情比名字听起来小得多。

## 本文依据

- https://nuxt.com/docs/4.x/api/commands/test（命令形状、"runs tests using `@nuxt/test-utils`"、Arguments 表 `ROOTDIR`、Options 表 `--cwd`/`--dev`/`--watch` 三项说明、两处 `NODE_ENV` 设为 `test` 的表述、指向 nuxt/cli 里 `commands/test.ts` 的 Source 链接）
- https://nuxt.com/docs/4.x/getting-started/testing（Running Tests 四条命令：全量、`--project unit`、`--project nuxt`、`--watch`）
