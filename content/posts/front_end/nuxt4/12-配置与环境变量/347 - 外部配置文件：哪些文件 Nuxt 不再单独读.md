---
url: /posts/6d428def94999d7086a476e50419823b/
title: 347 - 外部配置文件：哪些文件 Nuxt 不再单独读
date: 2027-02-04T11:00:00+08:00
updated: 2027-02-04T11:00:00+08:00
author: cmdragon

summary:
  摘要：很多前端项目习惯给每个工具单开一个配置文件。Nuxt 里有一类不再这么干：nitro.config.ts、postcss.config.js、vite.config.ts、webpack.config.ts 这四个 Nuxt 不读，配置必须写进 nuxt.config.ts 的对应键（nitro / postcss / vite / webpack）。官方 configuration 页用删除线把这几个文件名划掉、逐项指回 nuxt.config。但另一类工具（tsconfig、eslint、prettier、stylelint、tailwind、vitest）仍走各自的文件。这一篇把"哪些并入、哪些还在"分清楚。

categories:
  - 前端开发

tags:
  - nuxt4
  - 配置文件
  - vite
  - postcss
  - webpack
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


从别的项目过来的人，常会下意识在项目根建 `vite.config.ts`、`postcss.config.js`。在 Nuxt 里这些**不会被读到**。这一篇把官方 configuration 页的 "External Configuration Files" 讲清。

## 单一真源原则

官方定调：

> Nuxt uses `nuxt.config.ts` file as the single source of truth for configurations and skips reading external configuration files.

也就是 nuxt.config 是唯一真源，外部配置文件被**跳过不读**。

## 四个文件：划掉，改写进 nuxt.config 的键

官方那张表用删除线标了文件名，并把配置去向指回 nuxt.config：

| 工具 | 配置文件 | 怎么配 |
| --- | --- | --- |
| Nitro | ~~`nitro.config.ts`~~ | 用 `nuxt.config` 的 `nitro` 键 |
| PostCSS | ~~`postcss.config.js`~~ | 用 `nuxt.config` 的 `postcss` 键 |
| Vite | ~~`vite.config.ts`~~ | 用 `nuxt.config` 的 `vite` 键 |
| webpack | ~~`webpack.config.ts`~~ | 用 `nuxt.config` 的 `webpack` 键 |

这四个是重点。你放了 `vite.config.ts` 它不报错、就是被忽略——配置不生效、又没有明显提示，这是最容易困惑的地方（B5004 那条错误码就是专门针对"旁边放了个被忽略的独立配置"场景的）。正确做法是把内容搬进 nuxt.config 对应键、删掉外部文件。

## 另一些文件：仍然走各自的配置

同页还有一张"其他常见配置文件"表，这些是**保留独立文件**的，别一并搬走：

| 工具 | 配置文件 |
| --- | --- |
| TypeScript | `tsconfig.json` |
| ESLint | `eslint.config.js` |
| Prettier | `prettier.config.js` |
| Stylelint | `stylelint.config.js` |
| TailwindCSS | `tailwind.config.js` |
| Vitest | `vitest.config.ts` |

所以界线是"**Nuxt 会接管的那几个构建/服务端配置（nitro/vite/postcss/webpack）进 nuxt.config；纯编辑器/样式/lint/测试等工具配置仍在自己文件里**"。别把 `tsconfig.json` 也往 nuxt.config 塞（tsconfig 有它自己的规则，且 Nuxt 生成的是 `.nuxt` 里那份、你只在 nuxt.config 扩）。

## 怎么把配置写进对应键

举 vite 一个例子（configuration 页 Vue 配置节里就有）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  vite: {
    vue: {
      customElement: true,
    },
    vueJsx: {
      mergeProps: true,
    },
  },
})
```

原来 `vite.config.ts` 里的 `plugins`/`define`/`build` 等，现在都放在 `nuxt.config` 的 `vite` 键下面。webpack 同理（`webpack: { ... }`）。nitro、postcss 也一样——上一批 320 讲过 `nitro` 键就是 Nitro 配置的入口。

## 自查

- 别在 Nuxt 项目根放 `vite.config.ts`/`webpack.config.ts`/`postcss.config`/独立的 `nitro.config.ts`——Nuxt 不读它们；内容搬进 nuxt.config 对应键并删掉外部文件。
- 但 `tsconfig.json`、`eslint`、`prettier`、`stylelint`、`tailwind`、`vitest` 的配置**留在各自文件**，不要往 nuxt.config 搬。
- 若发现某个构建配置"写了没生效、也没报错"，先怀疑是不是建在了一个被忽略的独立文件里。

## 下一篇

下一篇 348 讲官方那张 runtimeConfig vs app.config 对比表：Hydrated/Bundled、环境变量、响应式、类型、每请求、HMR、非基本类型七行各说明什么。
