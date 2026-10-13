---
url: /posts/67bafeb005a00e571b2065ae3eeceeb3/
title: 582 - pages：说明句还写着 Nuxt 3，且没有类型没有默认值
date: 2027-04-24T09:00:00+08:00
updated: 2027-04-24T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## pages` 的说明句是 "Whether to use the vue-router integration **in Nuxt 3**."——4.x 的配置参考里这一行还留着上个大版本的措辞。这一节**没有 `**Type**` 行、没有 `**Default**` 行**，只给了两句规则和一个示例：不填值时**按 `app/pages/` 目录是否存在**决定启用，另外可以传一个 glob 或 glob 数组去只扫部分文件（示例是 `pattern: ['**/*/*.vue', '!**/*.spec.*']`）。布尔写法的依据不在本节，而在页面目录那一节那句 "To force the pages system, set `pages: true`"；**`pages: false` 这个写法在整套 4.x 文档里一次都没出现过。**

categories:
  - 前端开发

tags:
  - nuxt4
  - 文件路由
  - vue-router
  - glob匹配
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

不想让框架扫页面文件、或者只想扫某一部分，`pages` 这一节看起来是入口。翻过去先看两行：第一行告诉你它管的是 vue-router 集成要不要启用，第二行说可以传 glob。**至于能填 `true`、`false` 还是只能填对象，这一节没写。**

## 这一节的记载

`## pages` 全文两句加一个示例：

> Whether to use the vue-router integration **in Nuxt 3**. If you do not provide a value it will be enabled if you have a `app/pages/` directory in your source folder.
>
> Additionally, you can provide a glob pattern or an array of patterns to scan only certain files for pages.

- **没有 `**Type**` 行。**
- **没有 `**Default**` 行。**
- `**Example**` 只有一段：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  pages: {
    pattern: ['**/*/*.vue', '!**/*.spec.*'],
  },
})
```

两处措辞痕迹要指出：**说明句里的 "in Nuxt 3"** 是 4.x 文档里没清理干净的旧版本措辞（这一节别处都写 Nuxt 4 或版本号徽标）；另一处是句子里把目录写成 `app/pages/`，而**这个目录名本身是可改的**——`dir.pages` 那一节的类型是 `string`、默认值正是 `"app/pages"`，说明句是 "The directory which will be processed to auto-generate your application page routes."。**如果你改过 `dir.pages`，本节那句"有 `app/pages/` 目录就启用"里的路径名就不再是你的实际目录**，两节之间没有互指。

## 省略时的判定，和强制启用它的两条路

本节给的唯一规则是："**如果你不提供值**，那么只要源码目录里有 `app/pages/` 就启用。"这条规则的出处是页面目录那一节的那句 `::note`：为了减小包体积，这个目录是**可选的**——只写 `app.vue` 时 `vue-router` 不会被打进包里。

想反过来强制启用，那一节给的是两条路，本节一条都没写：

> To force the pages system, set `pages: true` in `nuxt.config` or have a `router.options.ts`.

于是这一节的形状可以确证的有两种：**对象形式**（本节示例的 `pages: { pattern: [...] }`）和**布尔 `true`**（目录页那句）。**`pages: false` 这种写法在整套 4.x 文档里找不到任何一处**——包括"想关掉路由系统"这个最自然的猜测。要关，文档给的思路是**让目录不存在**（那条 `::note` 的因果就是这么写的），或者按目录页那条改用 `app.vue` 单文件应用。

## pattern：两条 glob 的读法

示例里的数组顺序有意义，读法是"先include再exclude"：

- `'**/*/*.vue'` —— 匹配至少有一层目录的 `.vue` 文件。
- `'!**/*.spec.*'` —— 排除测试文件（`!` 前缀取反）。

**但这一节没有说清三件事**：不写 `pattern` 时默认扫什么、`!` 前缀是不是官方约定的语法（这是 node-glob 系工具的通行写法，本节没写依据）、以及这些模式相对哪个目录求值。

第二句里"可以传**一个 glob 或 glob 数组**"是唯一关于取值形状的线索，示例只演示了数组那种。

顺带一个衔接点：页面文件能有哪些扩展名，本节不讲，由 `## extensions` 那节决定（类型 `array`，默认六项 `.js`、`.jsx`、`.mjs`、`.ts`、`.tsx`、`.vue`），页面目录那一节也把"合法扩展名"链到了那一节。**注意示例里的 pattern 只匹配 `.vue`**，照它写会把 `.ts`、`.tsx` 写的页面排除掉——这是示例本身的取舍，不是默认行为。

## 自查

- `## pages`：**无 Type、无 Default**；两句规则 ＋ 一个 `pattern` 示例，说明句里还留着 "in Nuxt 3"。
- 省略不写时的判定条件是"源码里有 `app/pages/` 目录"，而这条规则的完整表述在页面目录那一节（目录可选、只写 `app.vue` 时不含 `vue-router`）。
- 确证过的写法只有两种：`pages: { pattern: [...] }` 与 **`pages: true`**（强制启用，另一条路是存在 `router.options.ts`）。
- **`pages: false` 无任何文档依据**；要"没有路由系统"，官方给的写法是让页面目录不存在。
- `pattern` 的默认行为、`!` 取反语法的依据、模式求值的基准目录：**三处均无记载**，要实测。
- 目录名可改（`dir.pages`，默认 `"app/pages"`），改了之后本节那句路径名不再对应你的项目。

## 下一篇

下一篇 583 把 `plugins` 与 `modules` 两个数组放一起讲。前者的类型行是 `array` 而**没有默认值行**，条目可以是字符串（结尾 `.client`／`.server` 就只在对应上下文加载）也可以是 `{ src, mode }` 对象；最要紧的是那条 `::callout`：**`~/plugins` 目录里的插件本来就自动注册，只有要定制顺序时才需要写进 `nuxt.config`，而且所有插件会按 src 路径去重。**

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## pages` 两句原文含 "in Nuxt 3" 措辞、无 Type/Default 的事实、`pattern` 两元素示例；`### dir.pages` 的 `string` 与默认 `"app/pages"`；`## extensions` 的 `array` 与六项默认）
- https://nuxt.com/docs/4.x/directory-structure/app/pages（开头 `::note`：目录可选、只用 `app.vue` 时不含 `vue-router`、"set `pages: true` … or have a `router.options.ts`"；页面扩展名指向 `api/nuxt-config#extensions` 的那句）
