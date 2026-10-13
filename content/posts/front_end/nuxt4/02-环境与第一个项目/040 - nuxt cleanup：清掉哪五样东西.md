---
url: /posts/ef9de6db005e96355cdcc0d79712b324/
title: 040 - nuxt cleanup：清掉哪五样东西
date: 2026-10-25T10:00:00+08:00
updated: 2026-10-25T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方对 nuxt cleanup 的定义是"移除常见的 Nuxt 生成文件与缓存"，并把它删的东西逐条列了出来，一共五处：你的构建目录（默认 .nuxt，除非 buildDir 另有指定）、.output、dist、node_modules/.vite、node_modules/.cache。注意第一条的措辞——它跟着 buildDir 的实际值走，不是死盯 .nuxt 这个名字。命令侧的接口极小：只有一个位置参数 ROOTDIR 与一个 --cwd 选项，既没有只清某一处的开关，也没有 dry-run，所以它的粒度就是"这五样一起删"。删完之后有两件必做的补救：类型要靠 nuxt prepare 或跑一次 dev 或 build 重新生成（.nuxt 被删等于编辑器与 typecheck 都失去依据），而想验证构建产物得重新 nuxt build（.output 没了就没有可 preview 的东西）。另外，如果你只是想清缓存后重启开发服务器，nuxt dev 面板里的 shift-r"清空缓存后重启"是成本更低的那条路。

categories:
  - 前端开发

tags:
  - nuxt4
  - 缓存
  - 清理
  - 命令行
  - 排错

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


"清一下缓存"这件事，官方给了一条专门的命令。它的接口小到只有一行，但**它删的东西比你以为的多**——所以这一篇的重点不是"怎么用它"，而是"删完要补什么"。

## 官方定义与那份清单

> `cleanup` 命令移除常见的 Nuxt 生成文件与缓存，包括：
>
> - 你的构建目录，也就是 `.nuxt`，**除非 `buildDir` 另有指定**
> - `.output`
> - `dist`
> - `node_modules/.vite`
> - `node_modules/.cache`

先看接口，它几乎是空的：

```bash [Terminal]
npx nuxt cleanup [ROOTDIR] [--cwd=<directory>]
```

| 项 | 官方说明 |
| --- | --- |
| `ROOTDIR` | 你 Nuxt 项目的根目录（默认 `.`） |
| `--cwd=<directory>` | 指定 Nuxt 项目的根目录 |

**就这两个。** 从这张表能直接读出三件实操事实：

1. **没有"只清某一项"的开关**——不能只删 `.output` 而留下 `.nuxt`。
2. **没有 dry-run**——它不会先告诉你它要删什么。
3. **没有 `--force` 之类的确认绕过**——它也不设这道门。

所以它的粒度就是"这五样一起删"，**接受这一点再用它**。

## 第一条清单里那句"除非 buildDir 另有指定"

这一句我单独拎出来，因为它纠正一个很常见的直觉：**cleanup 认的是"构建目录"这个概念，不是 `.nuxt` 这个名字。** 官方写的是"your build directory, which is `.nuxt` unless [`buildDir`](https://nuxt.com/docs/4.x/api/nuxt-config#builddir) says otherwise"——你把 `buildDir` 改成别的名，它跟着改后的值清。

（`buildDir` 这个配置项本身在 080、052 展开。）

## 五样东西各自是谁的产物、删了会怎样

这才是需要逐条看的部分。前三项是**项目里的目录**，后两项是**装在依赖里的缓存**。

**一、构建目录（`.nuxt`）**

谁生成：`nuxt dev`、`nuxt build`、`nuxt prepare`（035 抄过官方那句"这些类型存放在 `.nuxt` 目录里，在你运行开发服务器或构建应用时生成"）。

删了会怎样：**这是五样里唯一"删了会立刻影响你写代码"的一项。** `.nuxt` 里躺着那四份 tsconfig（`app`/`node`/`server`/`shared`），删掉之后：

- 编辑器的类型提示失去依据（031 讲过类型不是插件变出来的）；
- `nuxt typecheck` 检查的东西不完整（036 第二查的那条坑，这是它的人造版本）。

**补救：清完先 `npx nuxt prepare`，再跑任何类型相关的动作。** 顺序不能反。

另外，052 那页的 `.nuxt` 说明里有句话和这条命令正好呼应：

> 你不应改动其中的任何文件，**因为在运行 `nuxt dev` 时整个目录会被重新创建**。

**也就是说：删 `.nuxt` 在机制上是安全的**（它是可重建的生成物），代价只是"重建之前那一会儿没类型"。

**二、`.output`**

谁生成：`nuxt build`（037：build 创建 `.output`，里面是应用、服务器与全部依赖）。

删了会怎样：**没有可 preview 的东西了。** 029 那条 `nuxt preview` 的定义是"在跑完 `build` 命令之后预览你的应用"——产物被删，它就没输入了。

**补救：想再验一次生产产物，`npx nuxt build` 重新构建。**

**三、`dist`**

官方在清单里只写了 `dist` 这个词，**没有说明它是谁的产物**。我不替它编解释。你只需要知道：**如果你自己把构建输出配到项目根一个叫 `dist` 的目录，它会在这条命令的清除范围内**（这是我按"它在清单里"给的风险提示，不是文档结论）。

**四、`node_modules/.vite`**

官方在这里没展开，但 071 那节会看到它的来历（Vite 的缓存目录 `node_modules/.vite`）。删它的效果就是**让 Vite 重新做一遍依赖预构建**——第一次启动会慢一点，属于"该慢的时候就得让它慢"。

**五、`node_modules/.cache`**

注意这一项**覆盖面比你想的宽**。029 抄过：preview 把自己记录在 `node_modules/.cache/nuxt/preview`。

**所以 `nuxt cleanup` 会顺带把"本项目已经有一个 preview 在跑"这条记录删掉。** 后果不是 preview 坏了，而是**接管判断失真**——039 末尾我把这条联动写在那一篇里："明明有进程还在、命令却说没有"，就是这里来的。

## 什么时候该用，以及什么时候不该用

把上面的清单反过来读，就有判据了。

**适合用的场景**（共同点是"生成物本身可疑"）：

- 类型和代码明显对不上、或者报错信息引用了一个你已经删掉的页面；
- 构建产物行为诡异（053、040 这一串里反复出现的"先把生成物删干净再试一次"）；
- 交接/切分支之后，想要一份干净的重建。

**不该用的场景**：你只是想清 Vite 缓存然后继续开发。

这时**更便宜的一条路在 dev 面板里**：033 那张快捷键表中的 **`shift-r` = "清空缓存后重启"**（Restart with a cleared cache）。它只影响这一次开发服务器的启动，不会顺手删掉 `.output`、也不会破坏 preview 的记录。

**我的操作顺序建议**（依据来自 035、036 两页的依赖关系）：

```bash [Terminal]
npx nuxt cleanup
npx nuxt prepare
npx nuxt typecheck
```

跳过第二步直接跑第三步，就是你亲手复现 036 那个"四查之二"的坑。

## 自查

- [ ] `nuxt cleanup` 删的五样东西分别由哪三条命令产生？
- [ ] 它认的是 `.nuxt` 这个名字，还是"构建目录"这个概念？
- [ ] 清完之后，想让类型和检查恢复，第一步该跑什么？
- [ ] 你只是想清 Vite 缓存继续开发——该用这条命令还是面板上的 `shift-r`？
- [ ] 它为什么会让 preview 的"接管判断"失真？

## 下一篇

041 讲两个"不用知道端口"的命令：`nuxt curl` 靠 dev 写在构建目录里的 `nuxt.lock` 找到服务器，`nuxt task` 打的是 Nitro 任务——而任务默认根本不启用。

## 本文依据

- `api/commands/cleanup`（cleanup 移除常见 Nuxt 生成文件与缓存，清单五项：你的构建目录即 .nuxt 除非 buildDir 另有指定、.output、dist、node_modules/.vite、node_modules/.cache；用法 npx nuxt cleanup [ROOTDIR] [--cwd=<directory>]，位置参数 ROOTDIR 默认 .，选项仅 --cwd，无 dry-run 与选择性清理开关）
- `api/commands/prepare`（创建 .nuxt 目录并生成类型；CI 或 postinstall 场景有用）
- `guide/concepts/typescript`（Auto-generated Types：类型存放于 .nuxt 并在运行开发服务器或构建应用时生成，也可 nuxt prepare 手动生成；四份 tsconfig 覆盖 app/node/server/shared）
- `api/commands/build`（build 创建 .output 目录，内含应用、服务器与全部依赖并为生产准备好）
- `directory-structure/nuxt`（warning：不应改动其中任何文件，因为运行 nuxt dev 时整个目录会被重新创建；该目录应加入 .gitignore）
- `api/commands/dev`（交互式面板快捷键表中 shift-r 为 Restart with a cleared cache；dev 把自己记录在构建目录里的 nuxt.lock）
- `api/commands/preview`（preview 把自己记录在 node_modules/.cache/nuxt/preview；preview 用于在跑完 build 之后预览应用）
