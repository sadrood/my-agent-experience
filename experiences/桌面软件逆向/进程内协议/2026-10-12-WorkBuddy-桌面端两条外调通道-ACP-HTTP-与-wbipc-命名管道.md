---
title: WorkBuddy 桌面端两条外调通道：ACP(HTTP) 与 wbipc(命名管道)
domain: 桌面软件逆向/进程内协议
date: 2026-10-12
tags: ["workbuddy", "acp", "wbipc", "命名管道", "JSON-RPC", "SSE", "本地回环", "agent调用"]
author: my_agent
---

【经过】目标：从外部程序（非 GUI、非扫码）驱动 WorkBuddy 桌面端的 agent 干活。
两条通道实测可用：
1) ACP（Agent Client Protocol）over HTTP：agent runtime 监听 127.0.0.1:56737，路径 /api/v1/acp，对回环地址免鉴权。
   流程：POST /api/v1/acp/connect → {connectionId, sessionToken}；随后 POST /api/v1/acp，请求头带 acp-connection-id / acp-session-token，体为 JSON-RPC：initialize → session/new{cwd,mcpServers} → session/prompt{sessionId,prompt:[{type:'text',text}]}；响应是 SSE（event: message / data: {...}），含 session/update 事件：agent_message_chunk、agent_thought_chunk、tool_call、config_option_update、available_commands_update 等。
   实测：一条 prompt 让它用自带的 workbuddy-sheetagent 技能，在指定 cwd 生成 xlsx（它自己写 build.py、用它自己的 binaries/python/envs/default 跑 openpyxl），172s 完成，产物真实可读。
2) wbipc（桌面端外部 IPC）：端点/票据在 ~/.workbuddy/wbipc/endpoint.json（命名管道 + ticket）。协议是「客户端先开口」：连上后先发 session_hello{protocol_min/max, client_nonce, ticket_id=sha256(ticket)[:16]}，服务端回 session_challenge，再发 session_prove{client_proof=HMAC-SHA256(ticket, transcript)}，收 session_hello_ack 后可用 broker/GetPipe 枚举管道 → wb.entitlement(subscription.check)、wb.request(http.fetch)、wb.wecom-a2a(authorize)；http.fetch 能以桌面端身份发内部 HTTP 请求（例：GET /v2/activity/banner 返回业务 JSON）。
【失败点/关键教训】
- 被动等 challenge 的写法会静默挂起（服务端 15s 握手超时踢连接），必须客户端先发 session_hello。
- node 脚本用 process.exit() 会截断 stdout：把结果同步写文件，别依赖 stdout（重定向到文件常为空）。
- session/new 不会新建会话，会复用桌面端当前会话（回放 historyReplay 全部历史，新消息插进同一会话）——有污染用户对话的副作用，调用前要说明。
- session/new 返回里 configOptions 会给出可用模型列表、权限模式（default/plan/auto/bypassPermissions/fullAccess…）、沙箱开关；available_commands_update 给出斜杠命令表（含 /skills /doctor /gateway）。
【可复用步骤】
1. netstat 找回环监听端口 → 试 /api/v1/acp/connect 是否 200；
2. 用 SSE 解析（逐行 data: 前缀 JSON）提取 tool_call/agent_message_chunk 判断 agent 真实动作；
3. 长 prompt 用后台任务 + 完整流落盘，避免工具超时；超时设 ≥200s；
4. wbipc 侧永远走「先 hello 再 prove」的顺序，ticket/endpoint 每次开机刷新，需重新读 endpoint.json。
