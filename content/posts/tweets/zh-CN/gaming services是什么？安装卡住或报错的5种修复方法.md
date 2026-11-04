---
url: /posts/c9f1e6a8b0d2475f6a9c1e3b5d7f8a0/
title: gaming services是什么？安装卡住或报错的5种修复方法
date: 2026-11-04T13:00:00+08:00
lastmod: 2026-11-04T13:00:00+08:00
author: cmdragon

summary: gaming services 是 Windows 上运行 Xbox Game Pass 游戏必需的系统服务，安装卡住或报错时可用五种方法修复：重启服务、重装 Gaming Services、修复 Xbox 应用、清理缓存、检查系统组件。附完整操作步骤。

categories:
  - tweets

tags:
  - gaming services
  - Xbox
  - Windows
  - 游戏修复

---
Xbox 游戏一点开就弹"需要安装 gaming services"，然后安装又一直转圈？**Gaming Services 是微软在 Windows 10/11 上为 Xbox Game Pass 与商店游戏准备的系统级服务，没有它，很多 PC 版游戏根本启动不了**。装不上、装一半失败是高频问题，本文给出五种修复方法，从轻到重逐个试。

## 一、它是什么、为什么总出问题

Gaming Services 以系统组件形式从 Microsoft Store 分发，但它的安装走的是特殊通道（不同于普通应用），所以对系统状态很敏感：许可证服务异常、缓存损坏、系统更新中断，都会让它卡在安装界面或反复报 `0x80073d26` 之类的错误码。

## 二、五种修复方法（由轻到重）

### 方法 1：重启相关服务

1. `Win+R` 输入 `services.msc` 回车；
2. 找到 **Gaming Services** 与 **Gaming Services (x64)**；
3. 右键 → 重新启动；再试一次游戏。

### 方法 2：用 PowerShell 彻底重装 Gaming Services（最常用）

1. 开始菜单搜 PowerShell，右键"以管理员身份运行"；
2. 依次执行：

```powershell
Get-AppxPackage *gamingservices* | Remove-AppxPackage -allusers
```

3. 执行完打开 Microsoft Store，搜索 "Gaming Services" 重新安装。

### 方法 3：修复/重置 Xbox 应用

设置 → 应用 → 安装的应用 → 找到 Xbox → 高级选项 → 依次点"修复"，无效再点"重置"。Xbox 应用与 Gaming Services 是捆绑关系，应用坏了会连带服务装不上。

### 方法 4：清理商店缓存

`Win+R` 输入 `wsreset.exe` 回车，等待商店窗口自动弹出，缓存清理后重试安装。

### 方法 5：检查系统组件完整性

管理员命令提示符依次运行：

```
sfc /scannow
DISM /Online /Cleanup-Image /RestoreHealth
```

完成后重启电脑再装。系统文件损坏导致的安装失败，这一步是根治手段。

## 三、装好之后，顺手做两件事

1. **确认账号环境干净**：频繁换区下载容易触发许可异常，账号密码要用强密码——用 [密码生成器](https://tools.cmdragon.cn/zh/apps/password-generator) 生成后妥善保存；
2. **注册论坛/攻略站用小号邮箱**：用 [临时邮箱](https://tools.cmdragon.cn/zh/apps/temp-email) 收验证码，避免主邮箱被营销邮件轰炸。

## 四、常见问题

**Q：卸载 Gaming Services 会让游戏变快吗？** 不会。它是商店游戏的依赖组件，卸了游戏直接打不开；性能问题应从显卡驱动和后台进程排查。

**Q：PowerShell 重装后商店里搜不到 Gaming Services？** 先装/修复 Xbox 应用，Gaming Services 会随之自动重新出现。

**Q：还有别的游戏相关检测工具吗？** [CMDragon 在线工具箱](https://tools.cmdragon.cn/zh) 有手柄检测等游戏外设检测工具，插上即测无需安装。

## 免费好用的热门在线工具

- [CMDragon 在线工具 - 高级AI工具箱与开发者套件 | 免费好用的在线工具](https://tools.cmdragon.cn/zh)
- [密码生成器 - 应用商店 | 免费好用的在线工具](https://tools.cmdragon.cn/zh/apps/password-generator)
- [临时邮箱 - 应用商店 | 免费好用的在线工具](https://tools.cmdragon.cn/zh/apps/temp-email)
- [随机抽奖工具 - 应用商店 | 免费好用的在线工具](https://tools.cmdragon.cn/zh/apps/lucky-draw)
