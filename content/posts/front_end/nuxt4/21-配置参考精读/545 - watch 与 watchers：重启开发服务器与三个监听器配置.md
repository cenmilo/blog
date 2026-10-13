---
url: /posts/a0d4009d72151defb06d555e2b1a43c8/
title: 545 - watch 与 watchers：重启开发服务器与三个监听器配置
date: 2027-04-11T11:00:00+08:00
updated: 2027-04-11T11:00:00+08:00
author: cmdragon

summary:
  摘要：`watch` 的定位是**"The watch property lets you define patterns that will restart the Nuxt dev server when changed."**——改了这些模式匹配的东西，**开发服务器会重启**（不是热更新）。它的值是字符串或正则的数组，而且**基准写得很细：字符串要么绝对路径，要么相对 `srcDir`（以及每个 layer 的 `srcDir`）；正则则是拿"相对 `srcDir` 的路径"去匹配**。这一节**只有 Type，没有默认值**。`watchers` 则分三块：`chokidar`（官方示例给出 `ignoreInitial` 与 `ignorePermissionErrors`，两个默认都是 `true`）、`rewatchOnRawEvents`（说明是"收到这些事件类型会让监听器重启"，**该节连 Type 都没有**）、`webpack`（把 `watchOptions` 直接交给 webpack，其中 `aggregateTimeout` 默认 `1000`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 开发服务器
  - watch
  - chokidar
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

改了配置文件之外的某个文件，页面没更新；或者反过来，改一下就把整个 dev server 重启一遍，白等几秒。

这两件事分别归 `watch` 和 `watchers` 管。

## watch：改变化触发的是重启

定义句：**"The watch property lets you define patterns that will restart the Nuxt dev server when changed."**

注意用词是 **restart**（重启），不是热更新。所以这一项的效果是"这些文件变了，整个开发服务器重来"。

第二条说明把基准写得很具体，这是这节唯一容易配错的地方：

> It is an array of strings or regular expressions. Strings should be either absolute paths or relative to the `srcDir` (and the `srcDir` of any layers). Regular expressions will be matched against the path relative to the project `srcDir` (and the `srcDir` of any layers).

三个读数：

| 你写的值 | 匹配/解析方式 | 基准 |
| --- | --- | --- |
| 绝对路径字符串 | 直接就是路径 | — |
| 相对路径字符串 | 相对 **`srcDir`**，也包括每个 layer 的 `srcDir` | `srcDir`（不是 `rootDir`） |
| 正则 | 拿"相对 `srcDir` 的路径"去匹配 | 同上 |

**基准是 `srcDir` 而不是 `rootDir`**——这与上一篇 `ignorePrefix` 那节用 `app/pages/` 这种从根写起的路径不同。写 `watch` 时如果按 `rootDir` 去推，就会差一层 `app/`。

图层（layer）也被明写进两种情况里，意味着**多图层项目里每个 layer 的 `srcDir` 都在匹配范围内**。

- **类型**：`array`
- **默认值：这一节没有列。** 它属于"只有类型、没有默认"那一批，也就是说**不写时监听什么，文档没说**。

## watchers：三块配置各自交给谁

`watchers` 的定位句是 **"The watchers property lets you overwrite watchers configuration in your `nuxt.config`."**——它是"覆盖监听器配置"的总入口，下面三个子项各对接一个外部对象。

### `watchers.chokidar`

**"Options to pass directly to `chokidar`."**，并给出 **See** 指向 chokidar 仓库。该节下列出两项，都带默认值：

| 子项 | 类型 | 默认 |
| --- | --- | --- |
| `ignoreInitial` | `boolean` | `true` |
| `ignorePermissionErrors` | `boolean` | `true` |

两项都是 `boolean`、都是 `true`，**官方在这两行没给任何说明句**（只有类型与默认）。`ignoreInitial` 字面含义是"忽略初次扫描"，`ignorePermissionErrors` 是"忽略权限错误"——**但这两条解释是选项名的读法，文档没有写**，要按库的语义确认得看 chokidar 那份 API 文档（官方给的正是那个链接）。

### `watchers.rewatchOnRawEvents`

说明只有一句：**"An array of event types, which, when received, will cause the watcher to restart."**——收到列出的事件类型时，让监听器重启。

这一项**没有 `- **Type**`，也没有默认值**，甚至没有说明事件名从哪来。想用它只能按句子写数组，并用一次实际改动验证事件是否触发；**可接受的事件类型清单文档未给**。

### `watchers.webpack`

**"`watchOptions` to pass directly to webpack."**，**See** 指向 webpack@4 的 watch options 文档。下面列的一项是：

| 子项 | 类型 | 默认 |
| --- | --- | --- |
| `aggregateTimeout` | `number` | `1000` |

这行没有说明句，只有类型与 `1000`。它在 webpack 侧的含义要看那份文档（官方链接指的是 v4 的 `watchOptions` 页面）——**注意链接版本是 webpack@4**，这条细节决定了参数集是否与你在用的 webpack 一致。

## 一个前提：这三块不是同时生效

`watchers` 里同时出现 chokidar 与 webpack 两套选项，说明**监听器实现随构建器/场景不同**。官方在这一节没有写"哪个场景读哪一块"，但从两项各自的定位句能推出：`chokidar` 那项是"直接交给 chokidar"，`webpack` 那项是"直接交给 webpack 的 watchOptions"。

所以判断方式只能是：**你用的服务端/客户端构建器是什么，就配对应那一块**（`builder` 决定用 vite 还是 webpack，另有 `watch` 与 dev server 的行为）。**两份同时配会不会冲突，文档未说明。**

## 自查

- `watch` 触发的是 **dev server 重启**；值是字符串或正则的数组，**基准是 `srcDir`（含每个 layer 的 `srcDir`）**，本节没有默认值。
- `watchers.chokidar` 两项 `ignoreInitial`／`ignorePermissionErrors` 都是 `boolean`、默认都是 `true`，官方没给说明句，语义要看 chokidar 文档。
- `watchers.rewatchOnRawEvents` 只有一句"收到这些事件类型会让监听器重启"，**没有类型也没有默认**，可用事件名文档未给。
- `watchers.webpack` 交给 webpack 的 `watchOptions`，其中 `aggregateTimeout` 默认 `1000`；官方链接的是 webpack@4 的文档。
- 三块各自对接一个外部实现，哪个场景读哪块、同时配是否冲突，文档都没写。

## 下一篇

下一篇 546 读 `app` 节的 `head`：默认 JSON 里 `meta` 已经带 viewport 与 charset 两条、`link`/`style`/`script`/`noscript` 四个数组默认是空，而官方示例里有一句"这块很可能会变"。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## watch` 整节：restart the Nuxt dev server 定义句、数组与字符串/正则的基准两句、只有 Type `array` 而无 Default；`## watchers` 整节：overwrite watchers configuration 定位句、`### chokidar` 的 pass directly to chokidar 与 See 链接及其两项 `ignoreInitial`/`ignorePermissionErrors`（均 `boolean`、默认 `true`）、`### rewatchOnRawEvents` 的唯一说明句（无 Type/Default）、`### webpack` 的 watchOptions 定位句与 webpack@4 链接、`#### aggregateTimeout` 的 `number`/`1000`）
