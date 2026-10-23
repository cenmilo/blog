---
url: /posts/a2a9fc29c9c8210e2108cbe717f470d2/
title: 413 - 共享类型的作用域：shared 目录谁能看见
date: 2027-02-26T11:00:00+08:00
updated: 2027-02-26T11:00:00+08:00
author: cmdragon

summary:
  摘要：409 讲了"类型放哪个目录就只对那个上下文可见"，这一篇把 `shared/` 内部的**扫描规则**钉死——这才是"为什么我放在 shared 里还是认不出"的真答案。官方给的只有一条线：只有 `shared/utils/` 与 `shared/types/` 会被自动导入，**这两个目录里再嵌套子目录就不自动导**，除非把这些目录同时加进 `imports.dirs` 与 `nitro.imports.dirs`；其余位置一律手动用 `#shared` 别名引。官方那张目录树逐行标注了谁能自动导入：`shared/capitalize.ts`、`shared/formatters/lower.ts`、`shared/utils/formatters/upper.ts` 都不算，只有 `shared/utils/lower.ts` 与 `shared/types/bar.ts` 算。扫描方式与 `app/composables/`、`app/utils/` 完全一致。

categories:
  - 前端开发

tags:
  - nuxt4
  - shared目录
  - 自动导入
  - 类型作用域
  - hash-shared
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

"共享类型两端都能用"这话只对了一半——它还得**放在能被扫到的那一层**。这一篇把 `shared/` 的扫描边界逐行讲清。

## 只扫两层

官方 How Files Are Scanned 一段：

> Only files in the `shared/utils/` and `shared/types/` directories will be auto-imported. Files nested within subdirectories of these directories will not be auto-imported unless you add these directories to `imports.dirs` and `nitro.imports.dirs`.

两个要点：**只有 `shared/utils/` 与 `shared/types/`** 这两层里的文件被自动导入；**这两个目录下的子目录不算**，要算得把目录同时加进 `imports.dirs` **和** `nitro.imports.dirs`（两个上下文各自配，缺一个另一侧就看不见）。

## 官方那张标注表

`directory-structure/shared` 给的目录树，逐行照抄官方注释：

```bash [Directory Structure]
-| shared/
---| capitalize.ts        # Not auto-imported
---| formatters
-----| lower.ts           # Not auto-imported
---| utils/
-----| lower.ts           # Auto-imported
-----| formatters
-------| upper.ts         # Not auto-imported
---| types/
-----| bar.ts             # Auto-imported
```

| 文件位置 | 自动导入？ |
| --- | --- |
| `shared/capitalize.ts` | ❌ |
| `shared/formatters/lower.ts` | ❌ |
| `shared/utils/lower.ts` | ✅ |
| `shared/utils/formatters/upper.ts` | ❌ |
| `shared/types/bar.ts` | ✅ |

`shared/utils/formatters/upper.ts` 这行最容易误判：它在 `shared/utils/` **底下**，看着应该算，但官方明确标 Not auto-imported——因为它在子目录里。

官方还点明扫描口径的出处：

> The way `shared/utils` and `shared/types` auto-imports work and are scanned is identical to the `app/composables/` and `app/utils/` directories.

也就是说这套规则跟 151 讲过的 `app/composables`、`app/utils` 完全同一套，不是 `shared/` 特有。

## 扫不到的用 `#shared`

不在自动导入范围的文件，用 `#shared` 别名手动引（官方："This alias ensures consistent imports across your application, regardless of the importing file's location."）：

```ts
// 直接在 shared 目录下的文件
import capitalize from '#shared/capitalize'

// 嵌套子目录里的文件
import lower from '#shared/formatters/lower'

// utils 里再套一层的文件
import upper from '#shared/utils/formatters/upper'
```

三条官方示例正好对上上面那三行 Not auto-imported。**用别名引路径写全、不依赖扫描**，所以想保持"少配目录"的原状，用 `#shared` 比往 `imports.dirs` 里塞目录更省事。

## 共享的两端是 app 与 server

自动导入生效后，同一个 `shared/` 文件在应用侧和服务端侧都不用 import 就能用。官方给的那对示例：

```vue [app/app.vue]
<script setup lang="ts">
const hello = capitalize('hello')
</script>
```

```ts [server/api/hello.get.ts]
export default defineEventHandler((event) => {
  return {
    hello: capitalize('hello'),
  }
})
```

而共享的边界仍是 409 那条 `::important`：`shared/` 里的代码**不能 import 任何 Vue 或 Nitro 代码**。所以 `shared/utils` 放纯函数、`shared/types` 放两端都要的类型，这套目录才有意义。

## 类型目录的选择

配合 409 的分配表，写类型时的判断顺序是：

1. 只在 Vue 应用用 → `app/types/`
2. 只在服务端用 → `server/types/`
3. 两端都要 → `shared/types/`（**顶层**，别开子目录，否则要双配目录）
4. 想让增强对 app、shared、服务端路由三处都可见 → 官方口径是放 `shared/` 下的 `.d.ts`（见 055/396）

版本口径记一下：`shared/` 目录从 **Nuxt v3.14+** 才有。

## 自查

- 只有 `shared/utils/` 与 `shared/types/` **顶层**被自动导入；子目录不算，除非同时进 `imports.dirs` 与 `nitro.imports.dirs`。
- `shared/utils/formatters/upper.ts` 这种"在 utils 底下"的也不算——官方标注表逐行写明。
- 扫不到的用 `#shared/capitalize`、`#shared/formatters/lower`、`#shared/utils/formatters/upper` 手动引。
- 扫描方式与 `app/composables`、`app/utils` 相同；`shared/` 代码不能 import Vue 或 Nitro 代码。
- 类型放置：只 app→`app/types`、只服务端→`server/types`、两端→`shared/types` 顶层。
- `shared/` 自 v3.14+ 起可用。

## 下一篇

下一篇 414 收 395-414 那几篇：把 `typescript` 那一节从"键清单"转成"取舍决策"——`typeCheck` 三档怎么选、`shim` 什么时候才该开、`strict` 何时关。
