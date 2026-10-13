---
url: /posts/4e2ee21e7c1df5709b9d28b7d2cf73af/
title: 068 - layers 目录：自动注册、每层都要有 nuxt.config 与那条优先级阶梯
date: 2026-11-03T11:00:00+08:00
updated: 2026-11-03T11:00:00+08:00
author: cmdragon

summary:
  摘要：layers/ 目录用来在你的应用里组织并自动注册本地图层，官方列出的这一层能装的东西几乎覆盖整个应用结构——nuxt.config.ts（会与主配置合并）、app.config.ts、app/components/、app/composables/、app/utils/、app/pages/、app/layouts/、app/middleware/、app/plugins/、server/、shared/，而自动注册这条能力在 Nuxt v3.12.0+ 才可用、命名图层别名 #layers/[name] 指向每一层的 srcDir 则是 v3.16.0 引入。有一条 important 级别的硬要求：每个图层都必须有一个 nuxt.config.ts 文件才会被识别为有效图层，即使它是空的。优先级这一节是这一节最实用的部分，官方给的阶梯从高到低是三条：你的项目文件永远最高、~~/layers 目录里自动扫描到的图层按字母序排（Z 的优先级高于 A）、写在 extends 配置里的图层（第一项优先级高于第二项）；要控制顺序可以给目录加数字前缀（1.base/、2.features/、3.admin/），也可以不改目录名而把它们写进 extends，此时列出的第一项优先级最高，而任何你没有在 extends 里列出的 ~~/layers 图层会保持自己的字母序扫描位置、排在你在 extends 里列出的那些图层之下。

categories:
  - 前端开发

tags:
  - nuxt4
  - 图层
  - 优先级
  - extends
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


067 那张表里，图层（layers）是最容易被误解的一层：**它不是模块，是一个"能自带目录结构的半截应用"。** 这一篇讲三件事：怎么注册、必须有什么文件、**同名资源谁赢**。

## 定位：一个目录 = 一个可复用的半截应用

> 使用 layers/ 目录来在你的应用里组织并自动注册本地图层。
>
> `layers/` 目录允许你在你的 Nuxt 应用里组织并共享可复用的代码、组件、组合式函数与配置。**你项目里 `layers/` 目录中的任何图层都会被自动注册。**

> 在 `layers/` 里的每一个子目录都被当作一个独立的图层。**一个图层可以包含与标准 Nuxt 应用相同的结构。**

官方给的两个版本门槛要记住：

> （note）`layers/` 目录的自动注册**在 Nuxt v3.12.0+ 可用。**

> （note）命名图层别名是在 **Nuxt v3.16.0** 引入的。

**换句话说：023 那篇说"Node 22.x 或更新"只是能装；图层这套能力的可用性还要看 Nuxt 小版本。**（对照 016 讲的版本支持期——用到新特性前先确认你钉的版本。）

官方给的适用场景（tip 那一行）我照抄，因为它帮你判断"该不该用图层"：

> 图层很适合用来以**领域驱动设计（DDD）**组织大型代码库、创建可复用的**组件库**或**主题**、跨项目共享**配置预设**，以及隔离**后台管理面板**或**特性模块**这类关注点。

## 一条 important：每层都必须有自己的 `nuxt.config.ts`

> 每个图层**必须有一个 `nuxt.config.ts` 文件才会被识别为一个有效图层，即使它是空的。**

**这是"我建了 `layers/base/` 但什么都没发生"的第一顺位原因。** 空文件也算，但**不能没有**。

官方给的完整结构里，两个图层各带一份配置：

```bash [Directory structure]
-| layers/
---| base/
-----| nuxt.config.ts
-----| app/
-------| components/
---------| BaseButton.vue
-------| composables/
---------| useBase.ts
-----| server/
-------| api/
---------| hello.ts
---| admin/
-----| nuxt.config.ts
-----| app/
-------| pages/
---------| admin.vue
-------| layouts/
---------| admin.vue
```

**读这份树有两个细节：**

1. **图层内部用的是 `app/` 与 `server/` 这套标准结构**（050 那张地图原样适用）——**不是 `runtime/app/`**。067 那条 `runtime/app/` 硬要求是**本地模块**的规则，图层没有这一层。
2. `admin` 这个图层同时提供了 `pages/admin.vue` 与 `layouts/admin.vue`——**这正是 059 里 `appLayout: 'admin'` 那种"后台用一套壳"的组织方式。**

## 一层能装什么：官方那份清单

> 每个图层可以包含：

| 内容 | 官方说明 |
| --- | --- |
| `nuxt.config.ts` | 图层专属配置，**会与主配置合并** |
| `app.config.ts` | 响应式应用配置（055） |
| `app/components/` | Vue 组件（**自动导入**） |
| `app/composables/` | Vue 组合式函数（自动导入） |
| `app/utils/` | 工具函数（自动导入） |
| `app/pages/` | 应用页面 |
| `app/layouts/` | 应用布局 |
| `app/middleware/` | 路由中间件 |
| `app/plugins/` | Nuxt 插件 |
| `server/` | 服务端路由、中间件与工具 |
| `shared/` | 应用与服务端之间的共享代码 |

**第一行那个"会与主配置合并"是图层与模块的实质区别之一**：模块是"执行一段逻辑"，图层是"贡献一份配置 + 一批文件"。合并细节（`app.config.ts` 的图层合并策略、数组用 Function Merger）在 055 讲过。

## `#layers/[name]` 别名

> **指向每一层 `srcDir` 的命名图层别名会自动创建。你可以用 `#layers/[name]` 别名访问一个图层：**

```ts
// Access the base layer
import something from '#layers/base/path/to/file'

// Access the admin layer
import { useAdmin } from '#layers/admin/composables/useAdmin'
```

**这两行示例说明了别名指向的是那一层的 `srcDir`**（官方措辞 "Named layer aliases to the `srcDir` of each layer"）——所以 `#layers/admin/composables/useAdmin` 对应的是 `layers/admin/app/composables/useAdmin.ts` 那一层里 `srcDir` 之下的路径（051 讲过 `srcDir` 默认 `app`）。

**对照 065 的 `#server`、066 的 `#shared`：这三个别名都是"绕开相对路径地狱"的手段，而且都保证"无论导入方在哪都能一致"。**

## 优先级阶梯：这一节最实用

> 当多个图层定义了同一资源（组件、组合式函数、页面等），**优先级更高的图层胜出。图层按字母序排序，靠后的字母有更高的优先级（Z > A）。**

Getting Started 那一页给了完整三级（从高到低）：

> 1. **你的项目文件** —— 始终拥有最高优先级
> 2. **从 `~~/layers` 目录自动扫描到的图层** —— 按字母序排序（Z 的优先级高于 A）
> 3. **`extends` 配置里的图层** —— 第一项比第二项优先级高

官方给的那个"同一个 `Button.vue` 出现在三处"的例子，把结论说得很死：

```bash [Directory structure]
layers/
  1.base/
    app/components/Button.vue    # Base button style
  2.theme/
    app/components/Button.vue    # Themed button (overrides base)
app/
  components/Button.vue          # Project button (overrides all layers)
```

> 在这个例子里：
>
> - 如果只有那些图层存在，使用的是 `2.theme/Button.vue`（字母序更高）
> - 如果你的项目里有 `app/components/Button.vue`，**它覆盖所有图层**

**"项目文件永远赢"这一条，正好解释了官方那句 tip 的意图：**

> 这个模式很适合用来创建**带默认值的基座图层，它们可以被更具体的图层逐步覆盖。**

**也就是说：图层的设计意图是"可被逐步覆盖的层叠"，不是"必须生效的配置来源"。** 你要覆盖它，就在项目里写同名文件——**这条通道永远有效。**

## 两种控制顺序的办法

**办法一：给目录加数字前缀**

> 你可以通过给目录加数字前缀来控制顺序：

```bash [Directory structure]
layers/
  1.base/        # Lowest priority
  2.features/    # Medium priority
  3.admin/       # Highest priority (among layers)
```

**注意注释里那句 "Highest priority (among layers)"**：字母序里靠后 = 优先级高，**所以 `3.` 最大、最优先**（这与 060/061 那两处"编号只是控制执行顺序"的语义不同——**图层这里的数字直接对应优先级方向**）。

**办法二：不改目录名，写进 `extends`**

> 或者，你可以在 `nuxt.config` 里通过把它们引用进 `extends` 来控制你 `~~/layers` 目录的顺序，而不必重命名你的目录。**各项像其它 `extends` 图层一样排序，第一项优先级最高：**

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    '~~/layers/admin', // highest priority
    '~~/layers/features',
    '~~/layers/base', // lowest priority (among the listed layers)
  ],
})
```

**这里有一个必须注意的反向关系：目录前缀法里"数字大优先"，`extends` 列表法里"第一项优先"。**

> `~~/...`（推荐）与 `~/...` 两种别名形式以及相对路径（`./layers/admin`）都受支持。**任何你在 `~~/layers` 里存在但没有列出的图层，保持它自己的字母序自动扫描位置，排在你在 `extends` 里列出的那些图层之下。**

**这句是那张阶梯里最容易被漏的一条**：**没点名的图层会降级到 `extends` 列表之下**，但仍在字母序内部排序。

## 图层还能从哪来：`extends` 支持远端

`extends` 不只用于本地目录：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    // Extend from a local layer
    '../base',
    // Extend from an installed npm package
    '@my-themes/awesome',
    // Extend from a git repository
    'github:my-themes/awesome#v1',
  ],
})
```

**三种来源同一条数组：本地相对路径、已安装的 npm 包、git 仓库（带 `#v1` 这种 ref）。**

官方还给了私有仓库与命名两项：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    // per layer configuration
    ['github:my-themes/private-awesome', { auth: process.env.GITHUB_TOKEN }],
  ],
})
```

> （note）**如果没有指定分支，这会克隆 `main`。**

> （tip）你可以在图层来源旁边的选项里覆盖一个图层的别名。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  extends: [
    [
      'github:my-themes/awesome',
      {
        meta: {
          name: 'my-awesome-theme',
        },
      },
    ],
  ],
})
```

**这个 `meta.name` 决定的是 `#layers/[name]` 里那个名字**（别名一节说的"命名图层别名"）。

以及底层实现的一句：

> Nuxt 使用 unjs/c12 与 unjs/giget 来继承远端图层。

（021 讲 `create nuxt` 时官方也是用 giget 拉模板——**远端内容抓取这一层是同一套工具。**）

## 该用目录还是 `extends`：官方给的分界

> - **`~~/layers` 目录** —— 用于**属于你项目的本地图层**
> - **`extends`** —— 用于**外部依赖（npm 包、远端仓库）或你项目目录之外的图层**

**官方最后那个完整例子把三级阶梯合到了一起，值得照着核对：**

> 如果你同时有 `~~/layers/custom`，那么优先级顺序是：
>
> - 你的项目文件（最高）
> - `~~/layers/custom`
> - `../base`
> - `@my-themes/awesome`
> - `github:my-themes/awesome#v1`（最低）

**注意 `~~/layers/custom` 排在所有 `extends` 项之前**——这正是那条"没列出的保持字母序自动扫描位置、但排在 `extends` 列出的之下"的**反面情形**：**`custom` 没被 `extends` 点名，所以它留在第 2 级；被点名的三项在第 3 级内部按列表顺序排。**

## 自查

- [ ] 建了 `layers/base/app/components/` 但没有任何效果，官方那条 important 指向缺了什么？
- [ ] 图层内部为什么不用 `runtime/app/`（067 那条要求在这儿不适用）？
- [ ] 项目文件与图层同名组件，谁生效？
- [ ] 目录前缀法与 `extends` 列表法，"优先级高"分别对应数字大还是第一项？
- [ ] 你没在 `extends` 里点名的 `~~/layers` 图层，位置会在哪里？
- [ ] `extends` 一个 git 仓库时不写分支，官方说会克隆什么？

## 下一篇

069 讲 `test/` 目录：官方给的那份树里，只有 `test/nuxt` 与 `tests/nuxt` 会默认进入应用的类型上下文，而 `typescript.tsConfig.include` 里的路径基准是生成的 `.nuxt/tsconfig.json`，不是项目根。

## 本文依据

- `directory-structure/layers`（标题：使用 layers/ 目录组织并在应用内自动注册本地图层；该目录允许组织并共享可复用代码/组件/组合式函数与配置，项目内该目录中的任何图层都会被自动注册；note 自动注册在 Nuxt v3.12.0+ 可用；tip 图层适合用 DDD 组织大型代码库、创建可复用 UI 库或主题、跨项目共享配置预设、隔离 admin 面板或特性模块等关注点；Structure：layers/ 里每个子目录被当作独立图层、图层可包含与标准 Nuxt 应用相同的结构、important 每个图层必须有一个 nuxt.config.ts 文件才会被识别为有效图层即使是空的、给出 base 与 admin 两层的完整目录树；Automatic Aliases：指向每一层 srcDir 的命名图层别名会自动创建并可用 #layers/[name] 访问，示例 #layers/base/path/to/file 与 #layers/admin/composables/useAdmin；note 命名图层别名在 v3.16.0 引入；Layer Content 十一项清单（nuxt.config.ts 会与主配置合并、app.config.ts、app/components/ 自动导入、app/composables/、app/utils/、app/pages/、app/layouts/、app/middleware/、app/plugins/、server/、shared/）；Priority Order：多个图层定义同一资源时优先级更高的胜出、图层按字母序排序且靠后字母优先级更高（Z > A）、可用数字前缀 1.base/2.features/3.admin/ 控制顺序、也可在 extends 里引用目录（extends: ['~~/layers/admin','~~/layers/base']）以在不重命名的情况下排序且第一项优先级最高；read-more 指向 getting-started/layers#layer-priority）
- `getting-started/layers`（图层是 Nuxt 的核心特性之一、图层结构与标准 Nuxt 应用几乎相同；Use Cases 七条（用 nuxt.config 与 app.config 共享配置预设、用 app/components/ 做组件库、用 app/composables/ 与 app/utils/ 做工具与组合式函数库、创建 Nuxt 模块预设、跨项目共享标准设置、创建 Nuxt 主题、用模块化架构与 DDD 改善大型项目代码组织）；Usage：默认 ~~/layers 目录里的任何图层都会自动注册为项目图层、note 自动注册引入于 v3.12.0、并会为这些图层自动创建指向各自 srcDir 的命名别名（如 ~~/layers/test 可通过 #layers/test 访问）、note 命名图层别名引入于 v3.16.0；也可通过在 nuxt.config 添加 extends 属性继承图层并给出三种来源（'../base' 本地、'@my-themes/awesome' 已安装 npm 包、'github:my-themes/awesome#v1' git 仓库）；私有仓库可用 ['github:...', { auth: process.env.GITHUB_TOKEN }] 传认证 token；note 未指定分支时会克隆 main；tip 可在图层来源旁的选项里覆盖图层别名并给出 ['github:my-themes/awesome', { meta: { name: 'my-awesome-theme' } }] 示例；Nuxt 使用 unjs/c12 与 unjs/giget 继承远端图层；Layer Priority：更高优先级的图层在定义相同文件或组件时覆盖更低优先级；Priority Order 从高到低三级——你的项目文件永远最高、从 ~~/layers 自动扫描的图层按字母序（Z 高于 A）、extends 配置里的图层（第一项高于第二项）；Practical Example 用 1.base/app/components/Button.vue、2.theme/app/components/Button.vue 与 app/components/Button.vue 说明仅有图层时用 2.theme 的版本、项目里有同名文件则它覆盖所有图层；Controlling Priority：给目录加数字前缀 1.base（lowest priority）/2.features（medium）/3.admin（highest，among layers）、tip 该模式适合创建带默认值、可被更具体图层逐步覆盖的基座图层；也可不改目录名而把 ~~/layers 目录写进 extends 由 nuxt.config 控制顺序且第一项优先级最高（admin/features/base 三行注释），~~~/... 推荐与 ~/... 别名以及相对路径 ./layers/admin 都支持、任何存在于 ~~/layers 但未列出的图层保持其字母序自动扫描顺序并排在你在 extends 列出的图层之下；When to Use Each：~~/layers 目录用于属于你项目的本地图层，extends 用于外部依赖（npm 包、远端仓库）或项目目录之外的图层；Full Example 与"若同时有 ~~/layers/custom 则顺序为 项目文件 > ~~/layers/custom > ../base > @my-themes/awesome > github:...#v1"；示例卡片 Content Wind）
- `directory-structure/modules`（对照用：本地模块的应用侧文件必须放在 modules/your-module/runtime/app/ 才能正确类型检查——图层不适用这条）
- `directory-structure/app/layouts`（对照用：appLayout 路由规则可按路径集中指定布局，与 admin 图层提供 layouts/admin.vue 的组织方式配合）
- `api/commands/init`（对照用：giget 同样是 create nuxt 拉取远端模板所用的工具）
