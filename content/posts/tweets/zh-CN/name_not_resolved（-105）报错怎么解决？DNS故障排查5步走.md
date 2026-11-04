---
url: /posts/b8e0d5f7a9c1364e5f8b0d2a4c6e7f9/
title: name_not_resolved（-105）报错怎么解决？DNS故障排查5步走
date: 2026-11-04T12:30:00+08:00
lastmod: 2026-11-04T12:30:00+08:00
author: cmdragon

summary: ERR_NAME_NOT_RESOLVED（net::ERR_NAME_NOT_RESOLVED，错误码 -105）表示浏览器无法把域名解析成 IP，多数是 DNS 故障。按"先确认域名→刷新缓存→换公共 DNS→查 hosts→查路由器"五步排查，多数 5 分钟内解决。

categories:
  - tweets

tags:
  - name_not_resolved
  - DNS故障
  - 浏览器报错
  - 网络排查

---
打开网页突然弹出 `net::ERR_NAME_NOT_RESOLVED`（错误码 -105），别的网站却都能上？**这个报错的意思是：浏览器把域名翻译成 IP 的环节失败了**——不是网站"挂了"，多数是你这边的 DNS 出了问题。按下面五步排查，绝大多数情况 5 分钟内解决。

## 一、先看懂这个报错

你输入网址后，浏览器要先问 DNS 服务器："这个域名对应的 IP 是多少？"问不到答案，就报 -105。所以问题出在**域名解析链路**：域名本身、本机缓存、DNS 服务器、hosts 文件、路由器，五处任一环节断了都会报错。

## 二、五步排查法

### 第 1 步：确认域名本身没错

先检查拼写（常见：`.cm` 手滑打成 `.com`、多了空格）；再用 [域名查询工具](https://tools.cmdragon.cn/zh/apps/domain-finder) 确认域名是否已过期或被注销——域名过期后解析记录被删，全世界都打不开。

### 第 2 步：刷新本机 DNS 缓存

- Windows：命令提示符运行 `ipconfig /flushdns`；
- macOS：终端运行 `sudo dscacheutil -flushcache`；
- Chrome 地址栏输入 `chrome://net-internals/#dns` 点 Clear host cache。

刷完再试，很多时候这一步就好了。

### 第 3 步：换公共 DNS（最有效的修复）

默认 DNS 可能被劫持或故障。改用公共 DNS：

1. 网络设置 → 修改 DNS 服务器；
2. 填入 `223.5.5.5`（阿里）或 `119.29.29.29`（腾讯），备用填 `8.8.8.8`；
3. 保存后重新打开网页。

不确定当前 DNS 是否正常？用 [DNS 工具箱](https://tools.cmdragon.cn/zh/apps/dns-toolkit) 直接测试域名的解析记录，对比多个 DNS 的返回结果。

### 第 4 步：检查 hosts 文件

以前改过 hosts 指向旧 IP 的话，会一直解析到失效地址。打开 `C:\Windows\System32\drivers\etc\hosts`（macOS 在 `/etc/hosts`），删掉与该域名相关的行。

### 第 5 步：重启路由器 / 换网络验证

路由器 DNS 缓存异常也会导致全家打不开。重启光猫路由；仍不行就用手机流量开热点连电脑试——热点下能打开，问题锁定在宽带线路，联系运营商。

## 三、常见问题

**Q：只有某个网站打不开，别的都正常？** 大概率是该域名自身的解析记录问题，用 [DNS 工具箱](https://tools.cmdragon.cn/zh/apps/dns-toolkit) 查它的 A 记录是否还存在，等站长修复或反馈。

**Q：报错里还有 `-105` 这个数字是什么意思？** 它是 Chromium 系浏览器的网络错误码，专门对应"域名解析失败"，和 DNS 问题是同一件事。

**Q：怎么确认我的网络本身是通的？** 跑一次 [网速测试](https://tools.cmdragon.cn/zh/apps/speed-test)，能连通测速服务器说明线路正常，问题只出在解析环节。

## 免费好用的热门在线工具

- [DNS工具箱 - 应用商店 | 免费好用的在线工具](https://tools.cmdragon.cn/zh/apps/dns-toolkit)
- [域名查询工具 - 应用商店 | 免费好用的在线工具](https://tools.cmdragon.cn/zh/apps/domain-finder)
- [网速测试 - 应用商店 | 免费好用的在线工具](https://tools.cmdragon.cn/zh/apps/speed-test)
- [IP归属地查询 - 应用商店 | 免费好用的在线工具](https://tools.cmdragon.cn/zh/apps/ip-geolocation)
- [CMDragon 在线工具 - 高级AI工具箱与开发者套件 | 免费好用的在线工具](https://tools.cmdragon.cn/zh)
