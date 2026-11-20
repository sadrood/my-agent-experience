---
title: 文字小说任务优先 novelflow，视频/漫剧任务才走 toonflow/video_gen
domain: 写作/工具选择
date: 2026-11-20
tags: ["novelflow", "toonflow", "工具选择", "防混淆", "小说创作"]
author: my_agent
---

【触发规则（防混淆）】用户提"小说/写手平台/章节/大纲/签约"等文字创作词 → 优先 novelflow MCP（tools load mcp_novelflow_*，先 get_agent_guide）；只有"视频/短片/漫剧/分镜画面"才走 toonflow / video_gen 链路。两者都叫"xxflow"，别串。

【为什么会混淆】① toonflow 的 40+ 工具在系统提示里常驻可见，novelflow 工具要装载后才出现，规划时"眼见"的更容易被选中；② 历史任务以视频向为主，经验库早期全是 video_media 类经验，遇到"创作"惯性先想到视频链路；③ novelflow 当时 token 是占位符、没跑过完整任务，缺少成功使用的记忆强化。

【防呆措施】云经验库已存精炼指南（检索 novelflow 可召回）；本地 tools/local/novelflow_skill/ 备份 SKILL.md + 工具参数表；服务器 get_agent_guide 随时返回全文；~/.my_agent/mcp_servers.json 已有 novelflow 条目，agent 启动即可连。
