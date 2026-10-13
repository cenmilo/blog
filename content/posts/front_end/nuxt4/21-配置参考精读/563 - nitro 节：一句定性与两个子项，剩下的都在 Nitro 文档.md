---
url: /posts/2f30a9469696654ebb92735874b83aae/
title: 563 - nitro 节：一句定性与两个子项，剩下的都在 Nitro 文档
date: 2027-04-17T11:00:00+08:00
updated: 2027-04-17T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：在 `api/nuxt-config` 里，`nitro` 这一节的**全部内容**是：定性句 **"Configuration for Nitro."**、一个指向 **Nitro configuration docs** 的链接，以及两个子项——`routeRules`（只写 **Type: object**，没有说明句也没有默认值）与 `runtimeConfig`（`object`，默认值是那段 `public: {}` ＋ `app` 四键 ＋ `nitro.envPrefix: "NUXT_"` 的 JSON）。也就是说：**Nuxt 的配置参考不替你列 Nitro 的可配范围**，它只声明"这个键透传"。要判断某个 `nitro.*` 选项能不能写，只能去 Nitro 的文档，或在 `nuxt.config` 里试一次看构建是否接受。

categories:
  - 前端开发

tags:
  - nuxt4
  - nitro
  - 配置
  - env-prefix
  - 透传
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`.output/server/index.mjs` 是 Nitro 产的，路由规则的匹配也是 Nitro 做的。于是很自然的一个问题：Nitro 的配置在 Nuxt 文档里写在哪？

答案短得出人意料。

## 这一节的全部

`## nitro` 在配置参考里长这样（逐字）：

> Configuration for Nitro.
>
> **See**: Nitro configuration docs

然后两个子项：

| 子项 | 官方记载 |
| --- | --- |
| `routeRules` | 只有 `- **Type**: object` 一行，无说明句、无默认值 |
| `runtimeConfig` | `- **Type**: object` ＋ 一份默认 JSON |

`runtimeConfig` 那份默认值：

```json
{
  "public": {},
  "app": {
    "buildId": "4a2e2d30-418f-41df-8e58-ed5df06de7fd",
    "baseURL": "/",
    "buildAssetsDir": "/_nuxt/",
    "cdnURL": ""
  },
  "nitro": {
    "envPrefix": "NUXT_"
  }
}
```

**这一节到此结束。** 三个后果：

**一、`nitro` 是透传口，不是清单。** Nuxt 侧只保证"这个键存在"，不承诺列出 Nitro 支持的选项。想知道能写什么，配置参考帮不上。

**二、`routeRules` 在这里连一句说明都没有。** 它的语义、九个可用属性全在渲染概念页（下一篇按那份表讲）。**同一份配置参考里，`## routeRules` 那个顶层节也只有三行**——两处都指回别处。

**三、`nitro.envPrefix` 的默认值就写在这行 JSON 里**：`"NUXT_"`。这是环境变量前缀的出处，也是"不带 `NUXT_` 前缀的变量不会进 `runtimeConfig`"这条规则的配置面。

## 为什么这份默认值出现在两处

配置参考里有两份几乎一样的 JSON：顶层 `## runtimeConfig` 的默认值也是 `public: {}` ＋ `app` 四键，但**没有 `nitro.envPrefix` 那一行**。

两处差别就是这两行 JSON 的差别：`nitro` 下面那份多了 `envPrefix`。这暗示 `nitro.runtimeConfig` 是"Nitro 侧看到的运行时配置形状"，而顶层 `runtimeConfig` 是"Nuxt 侧你写的那个对象"。**两者为什么长得不一样、是否等价合并，配置页没有任何解释**——所以想区分"我给 Nitro 用还是给 Nuxt 用"，只能靠 `useRuntimeConfig()` 与 Nitro 侧 `useRuntimeConfig()`（`nuxt/server` 那份）各读一次来看实际值。

## 唯一的透传警告在哪

Nuxt 侧对"往 `nitro` 里写东西"这件事，只有一句风险提示，而它不写在配置参考，写在服务端目录那一节的 Nitro Config 小节：

> This is an advanced option. Custom config can affect production deployments, as the configuration interface might change over time when Nitro is upgraded in semver-minor versions of Nuxt.

配套那句是 **"You can use `nitro` key in `nuxt.config` to directly set Nitro configuration."**，示例是空对象 `nitro: {}` 加一行注释 `// https://nitro.build/config`。

把三处并起来读，能得到的安全姿势是：**`nitro` 是官方承认的透传入口 ＋ 官方承认它不受 Nuxt 的接口稳定承诺保护**。所以只写必需项，并在升级 Nuxt 的次版本时把这一节单独复查一遍。

## 想确认某个 nitro 选项是否被接受

由于 Nuxt 侧不列清单，最实际的检验是构建一次看行为，而不是找文档。三步：

1. 在 `nitro` 下写那个键（例如一个 `prerender` 相关的开关）。
2. 跑 `npx nuxt build`（要涉及预渲染就 `npx nuxt build --prerender`）。
3. 看**结果**是否符合预期：产物 `.output/public/` 里有没有多出生成的文件，或 `nitro.build` 阶段日志有没有提示未知配置。

**这三步是本系列给的验证方法，配置参考在 `nitro` 这一节没有任何 Example。** 它能回答的问题有限：透传是否生效、值是否被 Nitro 读——**"Nuxt 会不会在未来某个次版本改掉这个键的形状"，只有那句警告能提醒，没有清单能查。**

## 自查

- `## nitro` 只有"Configuration for Nitro."一句 ＋ 一个外链 ＋ 两个子项（`routeRules` 只有类型、`runtimeConfig` 有类型与默认 JSON）。
- `nitro.envPrefix` 默认 `"NUXT_"` 就写在这份默认 JSON 里，它是环境变量前缀规则的配置面。
- 与顶层 `runtimeConfig` 的默认相比，这份多了 `nitro` 一段；两者关系官方未解释。
- 官方对透传的唯一提示在 `directory-structure/server`：高级选项、会影响生产部署、Nitro 配置接口可能随 Nuxt 次版本变化。
- 想知道某个 `nitro.*` 是否生效，只能构建一次看产物与日志，Nuxt 文档不提供清单。

## 下一篇

下一篇 564 讲那批"要写进 `nitro`、但记载在别处"的键：`nitro.prerender` 的 `concurrency` 与 `failOnError`、`nitro.storage` 的挂载写法，以及 `nitro` 与 `routeRules` 分属两处时该改哪一个。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## nitro` 整节：定性句、See 链接、`### routeRules` 仅 `object`、`### runtimeConfig` 的 `object` 与含 `nitro.envPrefix: "NUXT_"` 的默认 JSON；对照的 `## runtimeConfig` 默认 JSON 不含 `envPrefix` 与顶层 `## routeRules` 三行）
- https://nuxt.com/docs/4.x/directory-structure/server（Nitro Config 小节：`nitro` 键透传句、`nitro: {}` 示例与影响生产部署的 `::warning`）
- https://nuxt.com/docs/4.x/api/composables/use-runtime-config（`app` 四键与 `nitro.envPrefix` 的运行时读取面）
