---
url: /posts/cd5c237284a421b8b5def19545cffb37/
title: 345 - 配置从哪来：nuxt.config、.nuxtrc 与环境变量的优先级
date: 2027-02-04T09:00:00+08:00
updated: 2027-02-04T09:00:00+08:00
author: cmdragon

summary:
  摘要：同一个配置值，可能写在 nuxt.config、项目根 .nuxtrc、家目录全局 .nuxtrc，或作为环境变量——谁说了算？这一篇按官方文档把优先级摆出来：nuxt.config 覆盖 .nuxtrc，项目 .nuxtrc 覆盖全局 .nuxtrc（nuxt.config 压过两者）；而 runtimeConfig 那部分值在运行时还会被对得上名字的 NUXT_ 环境变量再覆盖一层。顺带说清 .nuxtrc 是 rc9 扁平语法（ssr=false、modules[]=、setups. 是 Nuxt 记账区别手改）、全局 .nuxtrc 在 macOS/Linux 是 ~/.nuxtrc、Windows 是 C:\Users\{username}\.nuxtrc。

categories:
  - 前端开发

tags:
  - nuxt4
  - 配置优先级
  - nuxtrc
  - 环境变量
  - nuxt.config
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

340-364 到这把"配置能写在哪、谁盖谁"收个全景。先给结论顺序，再逐层说依据。

## 优先级从高到低

把文档里的分散说法拼成一条链：

```
nuxt.config.ts  >  项目 .nuxtrc  >  全局（家目录）.nuxtrc
                                  [运行时的 NUXT_ 环境变量再覆盖 runtimeConfig]
```

依据一句一句来。

## 层一：nuxt.config 是单一真源

configuration 页对 nuxt.config 的定位：

> The `nuxt.config.ts` file is located at the root of a Nuxt project and can override or extend the application's behavior.

nuxtrc 页直接写明和 `.nuxtrc` 的关系：

> If present, the properties in the `nuxt.config` file will overwrite the properties in `.nuxtrc` file.

所以 **nuxt.config 压过 .nuxtrc**。configuration 页还有一句"single source of truth"式的话（"Nuxt uses nuxt.config.ts as the single source of truth for configurations"，见 347），呼应它优先。

## 层二：项目 .nuxtrc > 全局 .nuxtrc

nuxtrc 页 "Global .nuxtrc File" 给了完整链：

> This global `.nuxtrc` file allows you to define default settings that apply to all Nuxt projects on your system. However, project-level `.nuxtrc` files will override these global settings, and `nuxt.config` will take precedence over both.

三句话定序：全局 `.nuxtrc` 是你这台机器所有项目的默认；项目级 `.nuxtrc` 盖全局；`nuxt.config` 盖这两者。全局文件的位置官方也给了：macOS/Linux `~/.nuxtrc`、Windows `C:\Users\{username}\.nuxtrc`。

`.nuxtrc` 本身是 rc9 扁平语法：

```bash [.nuxtrc]
# Disable SSR
ssr=false

# Add Nuxt modules
modules[]=@nuxt/image
modules[]=nuxt-security

# Module setups (automatically added by Nuxt)
setups.@nuxt/test-utils="3.23.0"
```

- 简单标量/数组用这种扁平写（`ssr=false`、`modules[]=` 是追加）。
- `setups.` 那段是 Nuxt 自动加的记账区（追踪模块安装/升级状态），官方 note：应不手改（"should not be modified manually"）。
- 复杂配置官方 tip 让你用 `nuxt.config`（"For more advanced configurations, use nuxt.config."）。

## 层三：环境变量覆盖 runtimeConfig

上面是"配置项"的优先级。runtimeConfig 的**值**还多一重：运行时对得上名字的 `NUXT_` 环境变量会再覆盖它（335/342）。它不在"配置文件互盖"那条链上，而是专管 runtimeConfig 这一处、发生在运行时。所以一个 `runtimeConfig.public.apiBase` 最终值，先由 nuxt.config/.nuxtrc 定默认、再被 `NUXT_PUBLIC_API_BASE` 在运行时盖。

## 一张来源地图

| 你想配什么 | 写在哪 | 会被谁盖 |
| --- | --- | --- |
| 常规选项（ssr、modules、routeRules…） | nuxt.config（首选） | — （最高） |
| 简单标量/追加数组、本机默认 | .nuxtrc（项目 / 全局） | nuxt.config 盖项目、项目盖全局 |
| runtimeConfig 的密钥/基址默认 | nuxt.config 的 runtimeConfig | 运行时 NUXT_ 环境变量 |
| 模块安装/升级记账 | .nuxtrc 的 setups. | 别手改，Nuxt 维护 |

## 自查

- 优先级记这条链：nuxt.config > 项目 .nuxtrc > 全局 ~/.nuxtrc；外加"运行时 NUXT_ 变量覆盖 runtimeConfig 的值"。
- .nuxtrc 只适合扁平/简单/追加，复杂配置回 nuxt.config；setups. 是 Nuxt 记账别手改。
- 全局 .nuxtrc 影响本机所有项目，项目级盖它。
- 别把"配置文件互盖"和"环境变量覆盖 runtimeConfig"混成一谈——后者只在运行时、只管 runtimeConfig。

## 下一篇

下一篇 346 讲 nuxt.config 里的环境覆盖后缀 $production、$development、$env 与 --envName：给不同环境写不同配置块。

## 本文依据

- .nuxtrc（`directory-structure/nuxtrc`）：定位句 + rc9；"For more advanced configurations, use nuxt.config."；扁平示例（ssr=false / modules[]= / setups.@nuxt/test-utils）；"If present, the properties in the nuxt.config file will overwrite the properties in .nuxtrc file."；setups 记账 note "should not be modified manually"；Global .nuxtrc（macOS/Linux `~/.nuxtrc`、Windows `C:\Users\{username}\.nuxtrc`、"project-level .nuxtrc override global, nuxt.config take precedence over both"）。
- Configuration（`getting-started/configuration`）：nuxt.config 是项目根、覆盖/扩展行为；single source of truth 表述（见 347）；runtimeConfig 被环境变量覆盖（见 342/335）。
