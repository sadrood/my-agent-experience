---
title: WorkBuddy 更新被"安装目录存在用户项目目录"拦截：真因是安装目录里多出的文件夹
domain: 软件运维/Windows 应用更新
date: 2026-10-12
tags: ["WorkBuddy", "Electron", "更新失败", "安装目录", "逆向", "asar", "unicode转义"]
author: my_agent
---

## 经过
用户贴出 WorkBuddy（腾讯 Electron 桌面端，装在 D:\workbuddy）更新被拦的弹窗：
「检测到应用安装目录下存在用户项目目录，更新会导致项目记录丢失。请先将项目迁移到其他目录(非安装目录)后再重试更新。」
排查发现：该弹窗文案就在 app 的 main 进程 JS 里（不是独立 updater），关键代码：
  APP_KNOWN_DIRS = {bin, locales, resources, tools, log, logs, _, share-target, assets}
  isKnownAppDirectory(n) = 白名单 || n.startsWith('$') || n.startsWith('nsis')
  getInstallDir() = dirname(process.execPath)   // win32
  hasUserContentInInstallDir() = readdirSync(installDir,{withFileTypes:true})
      .some(e => e.isDirectory() && !isKnownAppDirectory(e.name))
  → 只要安装目录下出现**任意一个顶层文件夹**不在白名单里，就判定为"用户内容"，更新被拦
  （弹窗按钮「了解更多」→ codebuddy.cn 文档 FAQ：更新/重装/覆盖安装后安装目录下的工作空间消失）
本例真凶是上一次逆向分析任务落在安装目录里的产物目录 `D:\workbuddy\analysis`（以及 resources\app_unpacked）。

## 关键教训
1. 文案搜不到别急着下结论：Electron 打包 JS 里中文常被写成 \uXXXX 转义，直接按 UTF-8 字节搜"安装目录"是找不到的，要搜 "\u68C0\u6D4B" 这类转义串，或反过来在 JS 里搜变量名（updateBlocked / installDir / Guard）。
2. 定位这类"某 App 的弹窗是谁弹的"，最省力的顺序是：① 先上网搜弹窗原文（本次 Bing 一句原文就命中 WorkBuddy + 腾讯云问答，直接锁定应用）；② 再翻该 App 的 JS 找判定函数；③ 最后用同一套判定逻辑在本地"模拟"验证是否已解除。
3. 逆向/分析产物绝不能留在被分析软件的安装目录里——除了脏，还会触发这类"安装目录守卫"，把对方的自动更新彻底卡死。产物要落到自己项目的 output/ 下。
4. 该守卫是启发式的（只认顶层文件夹白名单），所以网上大量"误报"帖子本质都是安装目录里留了别的东西（解压目录、备份、临时文件夹）。

## 可复用步骤
1. 搜弹窗原文 → 锁定应用，拿到官方 FAQ 链接；
2. 解开 app 的 main 进程 JS（asar 提取），grep 变量名而非中文，找到判定函数与白名单常量；
3. 按同一逻辑本地复算：readdir(安装目录) 里的顶层文件夹是否都在白名单内；
4. 把多余目录**移出**（不是删）安装目录 → 保留到自己的 output/ 目录，再让用户重试更新；
5. 复算通过即为验证完成。
