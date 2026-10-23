---
url: /posts/cd032cf28a4d52710d0d5b6b6f191dda/
title: 476 - 报障前要收集什么：nuxt info 与复现步骤
date: 2027-03-19T11:00:00+08:00
updated: 2027-03-19T11:00:00+08:00
author: cmdragon

summary:
  摘要：官方报障页给的是四步，而且**第一步不是写复现，是先判断这件事该不该提 issue**：如果你是要人帮忙解决问题，官方明说"最好的方式是去 asking for help 而不是报 bug"。第二步是搜 open issues 与 discussions，找到相似的**在原帖下评论**而不是新开一条。第三步做最小复现，官方给的起点是 Nuxt sandbox、**只加必要的那部分代码**，还有一条 `::note`：**问题涉及 Vue 或 Vite 时先用 Vue SSR starter 试复现**。第四步判断责任层——官方写的是"报到最相关、最具体的地方，很可能是造成问题的 Nuxt 模块，或 Nuxt 依赖的上游库"。材料这一侧，`nuxt info` 一条命令就给出解析到的 Nuxt/Nitro/Vite 版本、包管理器、加载的模块与配置里的 build modules，脚本要读就加 `--json`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 报障
  - nuxt-info
  - 最小复现
  - 开源协作
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

你提了个 issue，贴上报错和二十行代码，一周没人动。多半不是没人看到，而是**这条 issue 让人没法开始查**。

官方报障那一节整节很短，四步里前两步就是用来避免这种结果的。

## 那一节的态度先摆明

开头两句解释了为什么要认真写报障：

> Even if you can't fix the underlying code, reporting a bug well can enable someone else with a bit more familiarity with the codebase to spot a pattern or make a quick fix.

报得好，价值在**让熟的人一眼看出规律或快速修掉**。

## 第一步：先分清是不是 bug

这一节的原文很短，但方向很重要：

> Consider if you're looking to get help with something, or whether you think there's a bug with Nuxt itself. If it's the former, we'd love to help you - but the best way to do that is through asking for help rather than reporting a bug.

**"要我帮忙"和"这是 Nuxt 的 bug"是两件事**，前者官方明确指向另一页（getting help）。绝大多数被晾着的 issue，其实是第一类被当成第二类提了。

## 第二步：先搜，然后回到原帖

> Search through the open issues and discussions first. If you find anything that seems like the same bug, it's much better to comment on an existing thread than create a duplicate.

两个搜索范围是 **open issues** 和 **discussions**（官方给的都是仓库里的链接）。第二句给的取舍很明确：**像同一个问题就在旧线程下评论，不要新开一条**——新开的那条大概率被关成重复。

## 第三步：最小复现，官方给了起点

这段是那一节里最长的一段，先给标准再给方法。

标准句里三个要求并列：

> It's important to be able to reproduce the bug reliably - in a minimal way and apart from the rest of your project.

**可靠复现、最小化、脱离你的项目本体**。它接了一句为什么：这样能缩小可能原因的范围，也让别人不仅找得到原因，还能**验证一个潜在修法**——这条常被忽略：没有复现，修的人连"有没有修好"都没法判断。

方法句给了起点：

> Start with the Nuxt sandbox and add the **minimum** amount of code necessary to reproduce the bug you're experiencing.

**从 sandbox 开始往里加代码**，而不是从自己项目里往外删——加到能复现就停。

还有一条 `::note` 是硬性指引，很多人不知道：

> If your issue concerns Vue or Vite, please try to reproduce it first with the Vue SSR starter.

**涉及 Vue 或 Vite 的问题，先用 Vue SSR starter 试一次复现**。这等于替你判断"这是框架层还是 Nuxt 层"。

官方在页面上摆了两组卡片入口，五份模板：

| 用途 | 名称 | 官方给的地址 |
| --- | --- | --- |
| Nuxt 复现 | Nuxt on StackBlitz | `https://nuxt.new/s/v4` |
| Nuxt 复现 | Nuxt on CodeSandbox | `https://nuxt.new/c/v4` |
| Vue 复现 | Vue SSR on StackBlitz | `stackblitz.com/github/nuxt-contrib/vue3-ssr-starter` |
| Vue 复现 | Vue SSR on CodeSandbox | `codesandbox.io/p/sandbox/github/nuxt-contrib/vue3-ssr-starter` |
| Vue 复现 | Vue SSR Template on GitHub | `github.com/nuxt-contrib/vue3-ssr-starter/generate` |

注意 Nuxt 那两个走的是 `nuxt.new`，路径里的 `v4` 就是 4.x——**别拿 v3 的 sandbox 复现 4.x 的问题**。

这一节的收尾句是最实用的一条动作：

> Once you've reproduced the issue, remove as much code from your reproduction as you can (while still recreating the bug).

**复现出来之后，再往下删，删到不能再删**。官方补了一句：在这上面花的时间会对修问题的人有巨大帮助。

## 第四步：判断责任在哪一层

> With a Nuxt project, there are lots of moving pieces - from Nuxt modules to other JavaScript libraries. Try to report the bug at the most relevant and specific place. That will likely be the Nuxt module causing an issue, or the upstream library that Nuxt is depending on.

官方的建议很直白：**报到最相关、最具体的地方，而那通常是造成问题的 Nuxt 模块，或者 Nuxt 依赖的上游库**——不是默认报到 Nuxt 仓库。

468 那篇讲 E1006 时其实给过一个现成的例子：`onPrehydrate()` 没被构建转换，根因在"某个未被转译的依赖"，修法也在你自己配置的 `build.transpile` 上——这一类问题既不该报 Nuxt，也不该只报模块。

## 材料：一条命令收齐版本信息

`nuxt info` 的用法行是：

```bash [Terminal]
npx nuxt info [ROOTDIR] [--cwd=<directory>] [--json]
```

参数与选项一共三项：

| 名称 | 说明 |
| --- | --- |
| `ROOTDIR` | 项目根目录（默认 `.`） |
| `--cwd=<directory>` | 指定项目根目录 |
| `--json` | 以 JSON 打印项目信息 |

它给什么的官方原话："the versions of Nuxt, Nitro and Vite it resolves, the package manager, the modules it loads and the build modules in its configuration"，`--json` 的用途写的是**"when you want to read it from a script"**。

四项内容正对着上面第四步：模块清单能告诉你装了什么、`build modules` 能看出构建器侧的模块。**Nuxt/Nitro/Vite 是"解析到的"版本**，在 monorepo 里这一点尤其关键——你以为的版本和实际解析到的版本可能不同（071 那篇的 `modulesDir` 也是同一类坑）。

还有一个顺手的路子：dev 的终端面板里按 `i` 进 info 视图，**在里面按 `shift-y` 复制的就是这份 `nuxt info` 表**（官方在 `nuxt dev` 那一节写明）。

## 报障前把六样东西备齐

（**这份清单是我按上面四步与 `nuxt info` 的能力整理成的，官方没有这么列过**。）

1. 一句话说清"做了什么、期望什么、实际发生什么"。
2. `nuxt info` 输出（或其 `--json`）。
3. 最小复现的分享链接——Nuxt 用 `nuxt.new/s/v4` 或 `nuxt.new/c/v4`。
4. 若涉及 Vue 或 Vite：Vue SSR starter 上能不能复现的结论。
5. 已搜过的 open issue / discussion 编号，或者"在原帖下补的评论链接"。
6. 你判断的责任层（模块／上游库／Nuxt 本体），并据此决定提到哪个仓库。

## 自查

- 四步的顺序是：是不是 bug → 搜旧帖 → 最小复现 → 判断责任层。前两步决定你的 issue 会不会被当成求助帖或重复帖处理。
- 复现要做成"可靠、最小、脱离你的项目"，起点是 sandbox 而不是从自己项目删；再复现之后还要继续删。
- 涉及 Vue 或 Vite 先用 Vue SSR starter 试，这一步同时是分层判断。
- 报到最具体的地方，往往是某个模块或上游库。
- `nuxt info` 报解析到的 Nuxt/Nitro/Vite、包管理器、加载的模块与 build modules；脚本读用 `--json`，dev 面板里 `i` 后 `shift-y` 可复制同一张表。

## 下一篇

下一篇 477 走一遍完整链路：接口里 `throw createError({ status: 500, data: {...} })`，到 `useFetch` 在 `error.value.data.data` 读到结构化信息，再让 `error.vue` 显示一句人话并能跳回安全页——每一步对应前面哪一篇的哪条规则。
