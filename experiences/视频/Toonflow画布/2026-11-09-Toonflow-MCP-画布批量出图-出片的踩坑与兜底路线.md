---
title: Toonflow MCP 画布批量出图/出片的踩坑与兜底路线
domain: 视频/Toonflow画布
date: 2026-11-09
tags: ["toonflow", "mcp", "画布", "文生图", "视频合成", "ffmpeg", "故障恢复"]
author: my_agent
---

【经过】用 Toonflow MCP 在画布上做「宫崎骏风格短片」：文本分镜 → 6 个图片生成节点（agnes-image-2.5-flash，1280x720）→ 视频节点 → 最后用分镜图做缓慢运镜合成 27s 成片。

【关键教训】
1) MCP 工具会整体消失：一次 mcp_toonflow_read 读 1.5MB 图片把会话/页面搞崩，之后所有 mcp_toonflow_* 报“未知工具”。恢复办法是重连，而不是反复重试工具。
2) 页面连接（connectionId）会掉：报 “Toonflow page has been disconnected”，要先 getAppState 拿新的 connectionId；若页面状态一直 directory=null / panel=workspace，直接 `systemctl restart toonflow-browser`（它是独立的 headless Chrome 服务，重启后自动回到 #/workspace 并重新登记 canvas 页面），比反复 switchPanel/switchCanvas 有效。
3) 视频模型不是都能图生视频：getSettings 里只有配了 key 的供应商可用（本例只有 agnes）。agnes-video-2.5-flash 只支持 mode=text；apiMart/compshare 支持 startFrame/singleImage 但没 key → 生成立刻返回 “Please configure the provider API Key”。mode 必须与已连接素材匹配，接了图片就不能用 text。
4) 失败任务会锁节点：旧任务 state=interrupted 时 setConfig/generate 都报“节点正在生成或删除”，必须先 cancelGeneration 才能重来。
5) 远端机器只有 2 核且被多人共用（load 30~48），在远端跑 ffmpeg zoompan(preset slow, 3200x1800) 会把 sshd 拖到 banner exchange 超时。教训：重活放本地做，远端只存取文件。
6) 传大文件用 scp 到 workspace/assets/，再用 videoNode 的 node:setVideo（path 用工作区相对路径）挂到画布。

【可复用步骤】
- 批量建节点：写一个走 MCP JSON-RPC 的脚本（initialize → notifications/initialized → tools/call），参数外层是 {args:{...}, target:{connectionId}}，addNode 结果在 result.structuredContent.details.node。
- 图片节点：node:setConfig(size/ratio/model) → node:setPrompt → node:generateImage → node:getGenerationStatus 轮询，6 张可并行。
- 视频不可用时：把分镜图交给本地 ffmpeg 做 zoompan 运镜（每镜 5s，xfade 0.6s 交叉溶解）+ 褐噪声低通做环境风声，比空等队列快得多。
- 收尾：fitCanvas 一屏看完，删掉失败的生成节点，把成片挂成 videoNode。
