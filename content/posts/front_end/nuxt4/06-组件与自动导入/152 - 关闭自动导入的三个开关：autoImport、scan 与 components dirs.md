---
url: /posts/1f7e8c6ce9c2d6aafa3a0aac6f67e0dd/
title: 152 - 关闭自动导入的三个开关：autoImport、scan 与 components dirs
date: 2026-12-01T11:00:00+08:00
updated: 2026-12-01T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：想把自动导入关掉，官方给的其实是**三个不同层级的开关**，各关各的范围。**第一个是 `imports.autoImport: false`**，官方标题就叫 "Disabling Auto-imports"："If you want to disable auto-importing composables and utilities, you can set `imports.autoImport` to `false` in the `nuxt.config` file."，后果句是"**This will disable auto-imports completely but it's still possible to use explicit imports from `#imports`.**"。**第二个是 `imports.scan: false`**，标题是 "Partially Disabling Auto-imports"："If you want framework-specific functions like `ref` to remain auto-imported but wish to disable auto-imports for your own code (e.g., custom composables), you can set the `imports.scan` option to `false`"，两条后果是"Framework functions like `ref`, `computed`, or `watch` will still work without needing manual imports." 与 "**Custom code, such as composables, will need to be manually imported in your files.**"；`nuxt-config` 那一节给的正面定义更精确："**Whether to scan your `app/composables/` and `app/utils/` directories for composables to auto-import. Auto-imports registered by Nuxt or other modules, such as imports from `vue` or `nuxt`, will still be enabled.**"（类型 `boolean`，默认 `true`）。这一项挂着一条官方 `::warning`，两条副作用必须知道："**If you structure your project with layers, you will need to explicitly import the composables from each layer, rather than relying on auto-imports.**" 与 "**This breaks the layer system's override feature.** If you use `imports.scan: false`, ensure you understand this side-effect and adjust your architecture accordingly."。**第三个在组件那一侧**："Nuxt also automatically imports components from your `~/components` directory, **although this is configured separately** from auto-importing composables and utility functions."，关法是 `components: { dirs: [] }`，并附一句"though **note that this will not affect components added by modules**"；而 `components` 的官方默认值是两项——`{ path: '~/components/global', global: true }` 与 `'~/components'`，也就是说**清空时这两个一起没了**。顺带一条常被误用的边界：`imports.dirs` 是"加"不是"换"——"**Note that this option will not override the default directories (`~/composables`, `~/utils`)**.**"

categories:
  - 前端开发

tags:
  - nuxt4
  - 自动导入
  - nuxt-config
  - imports
  - layers

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


"关掉自动导入"这句话其实有三种做法，效果差别很大：有人以为 `imports.autoImport: false` 之后组件也不能直接用了，也有人以为关掉扫描会连 `ref` 一起失去。这一篇把三个开关的范围一次对齐。

## 三个开关的定位

| 开关 | 官方小节标题 | 关掉的到底是什么 |
| --- | --- | --- |
| `imports.autoImport: false` | Disabling Auto-imports | auto-imports 机制本身（"composables and utilities"这一路） |
| `imports.scan: false` | Partially Disabling Auto-imports | **只关扫描你的目录**（`app/composables/`、`app/utils/`） |
| `components.dirs: []` | Auto-imported Components 一节内 | 组件自动注册（与上面那一路**分开配置**） |

三者不是包含关系：`scan` 是 `autoImport` 的一个子集，而 `components.dirs` 与 `imports` 那一路是两套。

## 开关一：`imports.autoImport`

官方原文两段：

> If you want to disable auto-importing composables and utilities, you can set `imports.autoImport` to `false` in the `nuxt.config` file.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    autoImport: false,
  },
})
```

> This will disable auto-imports **completely** but it's still possible to use **explicit imports** from `#imports`.

两句话各有重点：

- **"completely"**：这是唯一被官方标为"完全关闭"的那个。
- **`#imports` 仍然可用**：显式导入这条路不走自动注入，它走的是别名解析（150 引的那句"every auto-import"说明别名覆盖范围）。所以关了自动导入之后，你的代码变成这样：

```ts
// imports.autoImport: false 之后必须自己写
import { computed, ref } from '#imports'
```

顺带一个真实代价：Vue API 也要显式导。这一项和 150 里那条官方约定是配对的——"In the docs, every function that is not explicitly imported is auto-imported by Nuxt"，一旦 `autoImport: false`，**文档里的示例代码在你项目里就不再能直接粘贴运行**（105 讲过官方示例的书写惯例）。这是我建议慎用它的第一个理由。

## 开关二：`imports.scan`

官方动机句：

> If you want framework-specific functions like `ref` to remain auto-imported but wish to disable auto-imports for your own code (e.g., custom composables), you can set the `imports.scan` option to `false` in your `nuxt.config.ts` file:

```ts
export default defineNuxtConfig({
  imports: {
    scan: false,
  },
})
```

> With this configuration:
> - Framework functions like `ref`, `computed`, or `watch` will still work without needing manual imports.
> - Custom code, such as composables, will need to be manually imported in your files.

`nuxt-config` 那一节对同一个键给的定义更精确，值得并排读：

> Whether to scan your `app/composables/` and `app/utils/` directories for composables to auto-import. **Auto-imports registered by Nuxt or other modules, such as imports from `vue` or `nuxt`, will still be enabled.**

- **Type**: `boolean`
- **Default:** `true`

两节合起来能读出这个键的准确边界：**它管"扫你的目录"，不管"框架与模块注册的名字"**。所以 `ref`、`useFetch` 这类仍然免写 import——这与官方"framework functions still work"那句是一致的。

### 但它的两条副作用很重

官方用 `::warning` 标了出来，我认为这是这一篇最需要背下来的部分：

> **Caution:** This setup has certain limitations:
> - If you structure your project with layers, you will need to explicitly import the composables from each layer, rather than relying on auto-imports.
> - **This breaks the layer system's override feature.** If you use `imports.scan: false`, ensure you understand this side-effect and adjust your architecture accordingly.

两条都指向同一件事：**层（layer）提供的 composables 也是靠扫描注册的，而"覆盖"能力依赖于自动导入的名字解析顺序**。

这正好和 068 那条"每层都有 nuxt.config、上层可覆盖下层"的规则冲突：**一旦 `scan: false`，覆盖机制就失效**，你会得到"文件明明在层里、导不进来"与"同名 composable 覆盖了但没生效"两种表现。

我的判断标准很朴素：**项目里有 `layers/`（或装了第三方 layer），就不动 `imports.scan`**。这一条不是官方结论，而是把官方那两句副作用应用到架构上的直接推论。

## 开关三：`components.dirs`

组件这一路的独立性，官方专门用一句话提醒过：

> Nuxt also automatically imports components from your `~/components` directory, **although this is configured separately from auto-importing composables and utility functions.**

关法与那句附注：

> To disable auto-importing components from your own `~/components` directory, you can set `components.dirs` to an empty array (though **note that this will not affect components added by modules**).

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  components: {
    dirs: [],
  },
})
```

`components` 这一项在 `nuxt-config` 里的定位与默认值一起给出：

> Configure Nuxt component auto-registration.
> Any components in the directories configured here can be used throughout your pages, layouts (and other components) without needing to explicitly import them.

- **Type**: `object`
- **Default**

```json
{
  "dirs": [
    {
      "path": "~/components/global",
      "global": true
    },
    "~/components"
  ]
}
```

**默认值是两项**，这个事实决定了 `dirs: []` 的实际后果：

| 被清空 | 后果 |
| --- | --- |
| `'~/components'` | 你自己的组件不再自动注册（156 会讲组件名怎么从路径生成） |
| `{ path: '~/components/global', global: true }` | 那个"全局注册"目录也一并没了 |
| 模块添加的组件 | **不受影响**（官方那句 note） |

第二项很容易被忽略：如果你的项目把一些通用组件放在 `components/global/` 图省事，`dirs: []` 会让它们同时消失，症状是"页面里标签还在、组件解析失败"。

## `imports.dirs` 不能拿来"换掉"默认目录

这一条不算关闭，但方向常被误解，官方在同一个 `imports` 节里写明：

> An array of custom directories that will be auto-imported. **Note that this option will not override the default directories (`~/composables`, `~/utils`).**

```ts
export default defineNuxtConfig({
  imports: {
    // Auto-import pinia stores defined in `~/stores`
    dirs: ['stores'],
  },
})
```

所以：**`dirs` 是加法**。想让某个默认目录不再被扫，靠 `dirs` 做不到；能停掉这条扫描的是 `imports.scan`（它同时停掉 `app/composables/` 与 `app/utils/` 两个，不能只停一个——官方定义里这两个目录是一起写的）。

`imports` 节里还有一个 `global` 键，官方那一节**只给了类型与默认值**：

- **Type**: `boolean`
- **Default:** `false`

没有配任何说明句。这一项的作用域我不在本文里猜；要用的话，先按 153 那条线看 kit 侧 `addComponent` 的 `global` 字段（组件全局注册有明确定义），别把两处当成同一个东西。

## 三张对照表：关掉之后还剩什么

| 内容 | `autoImport: false` | `scan: false` | `components.dirs: []` |
| --- | --- | --- | --- |
| `ref`/`computed`/`watch` 免写 import | ❌（官方："disable auto-imports completely"） | ✅（"still work without needing manual imports"） | ✅ |
| `useFetch`/`useState`/`useRuntimeConfig` 等框架 composable | ❌ | ✅（"registered by Nuxt ... will still be enabled"） | ✅ |
| 模块注册的自动导入 | ❌ | ✅（原文点名"or other modules"） | 组件侧不受影响 |
| 你写的 `app/composables/`、`app/utils/` | ❌ | ❌（"will need to be manually imported"） | ✅ |
| `shared/utils/`（066、151） | ❌（官方："completely"） | **文档未写**：`scan` 的定义只提到 `app/composables/` 与 `app/utils/`，没提 `shared/`；官方另一处只说 `shared/utils`、`shared/types` 的扫描方式与那两个目录"identical" | ✅ |
| `~/components` 与 `~/components/global` 里的组件 | 不受这一项管 | 不受这一项管 | ❌ |
| 从 `#imports` 显式导入 | ✅（官方明确仍可用） | ✅ | ✅ |

**注意最后两行的分工**：组件与 composables 是两套系统，`imports` 下的两个键都关不掉组件自动注册。

## 该不该关：我的取向（非官方结论）

| 你的诉求 | 建议 |
| --- | --- |
| 想让依赖关系在文件头可见 | 别全局关，用 `#imports` 显式导入（150 那节，154 展开场合） |
| 团队里自动导入造成过命名冲突 | 先看 `priority`（151 那条裁决规则），关扫描是更大的动作 |
| 项目用了 layers | 不要动 `imports.scan`（官方明说会破坏覆盖） |
| 只想禁掉自己的组件自动注册（例如强制显式导入组件） | `components.dirs: []`，但要接受模块组件仍在、以及 `global` 目录一起被清掉 |
| 想彻底回到"全部手写 import" | `imports.autoImport: false`，并接受官方示例代码不能再直接粘贴 |

## 自查

1. 三个开关分别关掉什么？其中哪一条官方写的是"完全关闭"？
2. `imports.autoImport: false` 之后，`#imports` 显式导入还能用吗？依据是哪句？
3. `imports.scan` 的正面定义（`nuxt-config` 那一节）怎么说？它的类型与默认值是什么？
4. `scan: false` 之后，`ref` 与你自己写的 composable 分别是什么状态？官方那两行 bullet 怎么写的？
5. `scan: false` 对 layers 项目造成哪两条后果？其中"破坏覆盖"那一句的原文是什么？
6. `components.dirs: []` 与模块添加的组件之间是什么关系？
7. `components` 的默认值是几项、分别是什么？清空时那一项是容易被漏掉的？
8. 能不能用 `imports.dirs` 把 `~/composables` 换掉？官方哪句话否掉了这种用法？
9. 三条对照表里，为什么"显式导入 `#imports`"这一列全是 ✅？

## 下一篇

153 反过来讲"往里加"：`imports.dirs`、`imports.presets` 的字段、`imports:sources` / `imports:extend` / `imports:dirs` 三个构建期钩子，以及 kit 的 `addImports`、`addImportsDir`、`addImportsExtend` 各自加在哪一层。

## 本文依据

- `https://nuxt.com/docs/4.x/guide/concepts/auto-imports`
  - 原文（Disabling Auto-imports）："If you want to disable auto-importing composables and utilities, you can set `imports.autoImport` to `false` in the `nuxt.config` file." + `imports: { autoImport: false }`；"This will disable auto-imports completely but it's still possible to use explicit imports from `#imports`."（该句链接指向本页 `#explicit-imports` 小节）
  - 原文（Explicit Imports）："Nuxt exposes every auto-import with the `#imports` alias that can be used to make the import explicit if needed:" + `import { computed, ref } from '#imports'`
  - 原文（Partially Disabling Auto-imports）："If you want framework-specific functions like `ref` to remain auto-imported but wish to disable auto-imports for your own code (e.g., custom composables), you can set the `imports.scan` option to `false` in your `nuxt.config.ts` file:" + `imports: { scan: false }`；"With this configuration: Framework functions like `ref`, `computed`, or `watch` will still work without needing manual imports. Custom code, such as composables, will need to be manually imported in your files."；warning "**Caution:** This setup has certain limitations: If you structure your project with layers, you will need to explicitly import the composables from each layer, rather than relying on auto-imports. This breaks the layer system's override feature. If you use `imports.scan: false`, ensure you understand this side-effect and adjust your architecture accordingly."
  - 原文（Auto-imported Components）："Nuxt also automatically imports components from your `~/components` directory, although this is configured separately from auto-importing composables and utility functions."；"To disable auto-importing components from your own `~/components` directory, you can set `components.dirs` to an empty array (though note that this will not affect components added by modules)." + `components: { dirs: [] }`
  - 原文（读文档约定）："In the docs, every function that is not explicitly imported is auto-imported by Nuxt and can be used as-is in your code."
- `https://nuxt.com/docs/4.x/api/nuxt-config`
  - 原文（imports 节定位）："## imports" / "Configure how Nuxt auto-imports composables into your application."（See 链接指向 `directory-structure/app/composables`）
  - 原文（`imports.dirs`）："An array of custom directories that will be auto-imported. Note that this option will not override the default directories (`~/composables`, `~/utils`)." / Type `array`；示例 `dirs: ['stores']` 及注释 "Auto-import pinia stores defined in `~/stores`"
  - 原文（`imports.global`）："### `global`" 只给 "**Type**: `boolean`" 与 "**Default:** `false`"，无说明文字
  - 原文（`imports.scan`）："Whether to scan your `app/composables/` and `app/utils/` directories for composables to auto-import. Auto-imports registered by Nuxt or other modules, such as imports from `vue` or `nuxt`, will still be enabled." / "**Type**: `boolean`" / "**Default:** `true`"
  - 原文（components 节）："## components" / "Configure Nuxt component auto-registration." / "Any components in the directories configured here can be used throughout your pages, layouts (and other components) without needing to explicitly import them." / Type `object` / Default `{ "dirs": [ { "path": "~/components/global", "global": true }, "~/components" ] }`；See 链接指向 `directory-structure/app/components`
- `https://nuxt.com/docs/4.x/directory-structure/app/components` 与 `https://nuxt.com/docs/4.x/directory-structure/app/composables`
  - 对照用（见 062、063、156）：`components` 与 `imports` 两节的 See 链接目标
- `https://nuxt.com/docs/4.x/directory-structure/shared`
  - 对照用（见 151）："`shared/utils` 与 `shared/types` 的自动导入方式与 `app/composables/`、`app/utils/` **identical**"；子目录需要同时加进 `imports.dirs` 与 `nitro.imports.dirs`
