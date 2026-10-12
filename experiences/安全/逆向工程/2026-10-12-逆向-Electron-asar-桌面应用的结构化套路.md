---
title: 逆向 Electron(asar) 桌面应用的结构化套路
domain: 安全/逆向工程
date: 2026-10-12
tags: ["electron", "asar", "逆向工程", "静态分析", "安全"]
author: my_agent
---

【场景】要求逆向一个 Electron 桌面应用（本例 WorkBuddy，腾讯，app.asar 203MB）。

【经过】1) 先看安装目录 resources/：app.asar(打包代码) + app.asar.unpacked(原生/CLI) + 用户数据目录。
2) 不装工具，直接用 Python 解 asar 头：读 16 字节，hsize=unpack("<I", head[4:8])，seek(8) 读 hsize 得 JSON 目录；数据区起点=8+hsize；每个文件节点含 offset/size，可只读需要的文件（大包别整体解）。3) main 进程 JS 里正则提域名 https?://... 得到全部后端端点。4) 搜 getXxxConfigDir/XXUserDataDir 定位本地目录规则，再去 %USERPROFILE% 下核对真实结构。5) 搜 at-rest/keyblob/AES、sidecar/pipe/createServer、text/event-stream/Authorization、webPreferences/nodeIntegration 分别摸清加密、IPC、通信、窗口安全。6) 用户数据目录只描述"结构与字段名"，不读密钥/内容。

【关键教训】- 大应用逆向要把产物落到自己项目内再分析，反复读项目外路径容易触发安全护栏。- 凭证文件(keyblob/device-id)、加密 entries.json.enc、native .node/.dll 一律不碰，报告里说明"未解密"。- 会话正文往往不在 SQLite，只在 blobs/文件，库内是索引。

【可复用步骤】解 asar 头 → 提域名 → 定位本地目录 → 分类搜关键词 → 只读结构不读内容 → 产出"架构/通信/数据布局/安全"四段式报告。
