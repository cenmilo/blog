---
url: /posts/4a26f000e16a68b0e105e8c6abf99915/
title: nuxt add 与 nuxt module：装模块、加 layer、搜索与卸载
date: 2026-10-27T09:00:00+08:00
updated: 2026-10-27T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方说得很直白：nuxt add 就是 nuxt module add 的同一条命令，只是它把图层也一起接了（"It is the same command as nuxt module add, with layers included"）。跑一次 add 会发生三件事——用你的包管理器把包装成依赖（除非传 --skipInstall）、把它写进 package.json、并在 nuxt.config 里登记：模块写进 modules，图层写进 extends（除非传 --skipConfig）。模块与图层的登记位置不同，是这两页文档里唯一必须记住的分岔。搜索用 nuxt module search <QUERY>，它只列与你 Nuxt 版本兼容的模块并把命中的文字高亮，--nuxtVersion 可以显式过滤（默认自动检测）；卸载用 nuxt module remove，它会卸载包并从 nuxt.config 的 modules 数组里删掉；三个子命令都可以不带名字运行，此时变成交互式选择——add 从不兼容你版本的模块里挑、remove 从项目当前在用的模块里挑。另外官方明确：nuxt add <template> <name> 已废弃，脚手架文件要用 nuxt add-template。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块
  - 图层
  - 命令行
  - 包管理

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


009 那篇我讲过"模块、插件、图层是三个不同的东西"。这一篇讲**怎么把它们装进项目**——官方给了两条命令，但读下去你会发现它们其实是一条。

## `add` 与 `module add` 是同一条命令

官方在 `add` 页写了一句定性：

> `add` 命令把 Nuxt 模块与图层安装进你的应用。**它是 `nuxt module add` 的同一条命令，只是包含了图层。**

对照两页的用法行，确实**一字不差**：

```bash [Terminal]
npx nuxt add <MODULENAME...> [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--skipInstall] [--skipConfig] [--dev] [--packageManager=<npm|pnpm|yarn|bun|deno|aube|nub|upm>]
```

```bash [Terminal]
npx nuxt module add <MODULENAME...> [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--skipInstall] [--skipConfig] [--dev] [--packageManager=<npm|pnpm|yarn|bun|deno|aube|nub|upm>]
```

`module` 页那边也反过来指了一次：

> [`nuxt add`](https://nuxt.com/docs/4.x/api/commands/add) 是同一条命令，而且它也接受图层。

所以这一节只有一个结论：**记 `nuxt add` 就够了。** 但 `nuxt module` 还有另外两个子命令（`search`、`remove`）是 `add` 没有对应的，那部分才值得单独学。

## 跑一次 `add`，它动了哪三个地方

官方给的清单（这是本篇最该记住的一段）：

> 运行该命令时，它会：
>
> - 用你的包管理器把包安装为依赖，**除非你传 `--skipInstall`**
> - 把它加进你的 `package.json` 文件
> - 在你的 `nuxt.config` 文件里登记它——**模块登记在 `modules`，图层登记在 `extends`**，除非你传 `--skipConfig`

第三条就是那条必须记住的分岔：

| 你装的东西 | 写进 `nuxt.config` 的哪个键 |
| --- | --- |
| 模块 | `modules` |
| 图层 | `extends` |

**实操含义：装完没生效，先看登记位置对不对。** 图层被写进 `modules` 或者模块被写进 `extends`，都是"装了但框架不认"的表现。

选项表：

| 选项 | 官方说明 |
| --- | --- |
| `--cwd=<directory>`（默认 `.`） | 指定项目根目录 |
| `--logLevel=<silent\|info\|verbose>` | 构建期日志级别 |
| `--skipInstall` | 跳过 npm install |
| `--skipConfig` | 跳过 `nuxt.config.ts` 更新 |
| `--dev` | 把模块装成开发依赖 |
| `--packageManager=<npm\|pnpm\|yarn\|bun\|deno\|aube\|nub\|upm>` | 用哪个包管理器安装 |

`--skipInstall` 与 `--skipConfig` 这对组合，用途是**"我只想让命令改配置、不想让它现在装包"（或反过来）**——032 的 dev container 那种"镜像里已经装好依赖"的场景就适合 `--skipInstall`（这个联系是我按两页文档对上的，官方没有把这两个场景写在一起）。

## 模块是按**你项目的 Nuxt 版本**解析的

官方在举完多包同装的例子时带了一句重要的话：

> 可以一次添加多个包，**而模块会按照你项目所使用的 Nuxt 版本来解析**：

```bash [Terminal]
npx nuxt add pinia
```

```bash [Terminal]
npx nuxt add @nuxt/image @nuxt/fonts
```

这句"按你项目的 Nuxt 版本解析"在同一份文档里出现了三次，措辞略有不同但意思一致：

- `add` 页：不带名字运行时——"browse the available modules interactively"（浏览可用模块）
- `module add` 页：不带模块名时——"**从与你的 Nuxt 版本兼容的模块里挑**"
- `module search` 页：搜索结果——"与你使用的 Nuxt 版本**兼容**的模块"

**这一条能解释一个很常见的困惑：为什么某个模块在社区列表里有、你搜不到或装不上。** 按官方这三句，答案是版本兼容性过滤，不是它下架了。

## `nuxt module search`：搜的就是"兼容你这版"的模块

```bash [Terminal]
npx nuxt module search <QUERY> [--cwd=<directory>] [--nuxtVersion=<3|4|4.2.0>] [--json]
```

官方的说明里有两个细节值得抠：

> 该命令搜索匹配你查询条件的 Nuxt 模块，**这些模块要与你的 Nuxt 版本兼容**，并显示每个模块的描述，**其中命中的文字会被高亮**。

1. **高亮**是输出形态的一部分——它是给人眼看的交互界面（页面还配了一张终端截图）。
2. `--nuxtVersion=<3|4|4.2.0>` 可以显式按版本过滤，**默认是自动检测**（官方在默认列写的是 `auto detected by default`）。

`--json` 的用途官方写得很短："用 `--json` 拿到机器可读的 JSON 结果"——**要写脚本（例如给团队做模块白名单检查）就用它，别去解析带高亮的终端文本。**

## `nuxt module remove`：卸载 + 从数组里删

```bash [Terminal]
npx nuxt module remove [MODULENAME...] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--skipInstall] [--skipConfig]
```

> 该命令卸载这个包，并把它从你 `nuxt.config` 的 **`modules` 数组**里移除。

```bash [Terminal]
npx nuxt module remove pinia
```

这里有三处细节和 `add` 不对称，容易看漏：

- **`MODULENAME...` 在 remove 里是可选的**（用法行写成 `[MODULENAME...]`，而 add 写成 `<MODULENAME...>`）。不带模块名运行时，官方的说法是"从项目当前在用的模块里挑"。
- **remove 没有 `--dev`、也没有 `--packageManager`**（对照 add 的选项表）。
- **`--skipInstall` 在 remove 页的措辞变了**：add 页写"Skip npm install"，remove 页写"**Skip dependency uninstall**"。同一个开关名，在两条命令里指的是装与卸两个方向。

还有一点值得注意：remove 的说明里**只提到 `modules` 数组**，没提图层的 `extends`。**官方在这一页没写卸载图层要怎么做**——我就不替它编做法。

## 但别再用 `nuxt add <template> <name>`

这一条官方专门留了 note：

> `nuxt add <template> <name>` **已废弃**。请用 [`nuxt add-template`](https://nuxt.com/docs/4.x/api/commands/add-template) 来脚手架文件。

**这就是 `add` 与 `add-template` 的分界**（046 展开后者）：

| 你要做的事 | 命令 |
| --- | --- |
| 装第三方模块或图层（改依赖与配置） | `nuxt add` |
| 生成一个文件（组件、页面、插件、接口……） | `nuxt add-template` |

旧写法 `nuxt add component Foo` 官方说"仍然能跑，但已废弃"（add-template 页的措辞是 "which still runs but is deprecated"）。**新代码一律写 `add-template`。**

## 自查

- [ ] `nuxt add` 跑完之后，项目的哪三个文件被改过？
- [ ] 模块和图层分别登记在 `nuxt.config` 的哪个键下？
- [ ] 某个模块你搜不到，按官方三处措辞，最可能的原因是什么？
- [ ] `--skipInstall` 在 `add` 与 `module remove` 里各自指什么？
- [ ] 你要新建一个 `server/api` 下的接口文件，该用 `add` 还是 `add-template`？

## 下一篇

046 把 `add-template` 的十六种模板逐个摊开：每种生成在哪个目录、哪些带后缀修饰符（`.client`、`.get`），以及官方那条"名字会解析到项目外面就被拒绝"的安全限制。

## 本文依据

- `api/commands/add`（add 安装 Nuxt 模块与图层进应用、是 nuxt module add 的同一条命令且包含图层；参数 MODULENAME... 一个或多个以空格分隔；选项 --cwd 默认 .、--logLevel、--skipInstall（Skip npm install）、--skipConfig（Skip nuxt.config.ts update）、--dev（装为 dev dependencies）、--packageManager=<npm|pnpm|yarn|bun|deno|aube|nub|upm>；运行时的三个动作：用你的包管理器装为依赖除非 --skipInstall、加进 package.json、在 nuxt.config 里登记（模块进 modules、图层进 extends）除非 --skipConfig；示例 npx nuxt add pinia 与 npx nuxt add @nuxt/image @nuxt/fonts 且"modules will be resolved against the Nuxt version your project uses"；不带名字运行则交互式浏览可用模块；note nuxt add <template> <name> 已废弃请改用 nuxt add-template）
- `api/commands/module`（module add 用法与选项与 add 相同并给出 npx nuxt module add pinia 示例、不带模块名时从与你的 Nuxt 版本兼容的模块中挑选、note nuxt add 是同一条命令且也接受图层；module remove 用法行 [MODULENAME...] 可选与 --cwd/--logLevel/--skipInstall（Skip dependency uninstall）/--skipConfig、它卸载包并从 nuxt.config 的 modules 数组移除、示例 npx nuxt module remove pinia、不带名字时从项目当前使用的模块中挑选；module search <QUERY> 与 --cwd/--nuxtVersion=<3|4|4.2.0>（按 Nuxt 版本过滤仅列兼容模块，默认自动检测）/--json，它搜索与你 Nuxt 版本兼容的模块、显示每个模块描述并高亮命中文字、--json 得到机器可读输出）
- `api/commands/add-template`（add-template 为项目结构脚手架一个文件、它取代 nuxt add <template> <name> 而后者仍能运行但已废弃——用于本篇的分界）
