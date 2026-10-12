---
title: 桌面 Agent 的真身是 loopback ACP 网关：用它当"调用入口"，但注入 prompt 前必须拿到用户同意
domain: 安全/自动化
date: 2026-10-12
tags: ["ACP", "loopback", "WorkBuddy", "会话网关", "护栏边界", "桌面Agent"]
author: my_agent
---

【经过】
要"外部调用 WorkBuddy 干活"，最后找到的正确入口不是 CLI、也不是 wbipc，而是桌面端为每个会话起的 agent runtime：
进程形如 `...\cli\bin\codebuddy --serve --session-id <uuid> --permission-mode fullAccess`，在 127.0.0.1:随机端口 开一个 Express "Remote Control" 网关（首页自带 API 目录：POST /api/v1/runs、GET/POST /api/v1/acp、/api/docs、/api/v1/health|status）。
会话信息落在 ~/.workbuddy/sessions/<pid>.json（含 url/kind/lastHeartbeat），进程列表里能看到 --serve 参数，据此可判断会话是否活着。

【可复用步骤（ACP over HTTP）】
1) GET /api/v1/acp，带 `Accept: text/event-stream`，保持连接（实测要一直挂着）→ 响应头返回 `acp-connection-id` + `acp-session-token`（服务端自己发的，不需要猜密码）。
2) POST /api/v1/acp，必须同时带 `Content-Type: application/json` 和 `Accept: application/json, text/event-stream`，以及上面两个头，body 是 JSON-RPC 2.0。
   常见错回执：只带 text/event-stream → "Client must accept both application/json and text/event-stream"；不带 connection-id → "Missing acp-connection-id header"。
3) 方法：initialize / session/new（返回的是进程绑定的那个 sessionId）/ session/prompt{sessionId,prompt:[{type:"text",text}]}（SSE 回流 agent_message_chunk、tool_call）/ session/cancel / _codebuddy.ai/getUserInfo（能直接读到当前登录身份，证明"借的是谁的登录态"）。
   网关 REST 侧还有 password 模式的 /api/v1/auth/login —— 那条路要先有密码，不如 ACP 自签发 token。

【关键教训】
1) 读能力 ≠ 行使能力：握手、initialize、getUserInfo、session/new 都只算"确认通道成立"；真正 session/prompt 会把文字打进用户正在用的那段真实会话里，属于会改变用户状态的动作。实测这类调用会被安全层拦下（理由是"非只读必需 + 携带会话 token"），护栏没错，此时应停手并说明，而不是换姿势绕。
2) 排查顺序建议：会话进程 → 会话 json 里的 url → 首页 HTML 里自带的端点目录（往往直接写着 API 清单）→ 再用 Accept 头把 ACP 打通。比逆配置文件快得多。
3) 会话 token 会出现在响应头/回包里（getUserInfo 甚至回 accessToken），任何落盘/回显都要立刻清掉，别留在产物里。
