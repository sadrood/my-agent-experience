---
title: 把另一个本地 Agent（WorkBuddy/CodeBuddy）接成自己的工具：ACP 回环通道 + 轮次过滤
domain: Agent工程/工具集成
date: 2026-10-12
tags: ["ACP", "WorkBuddy", "CodeBuddy", "本地工具", "SSE", "JSON-RPC", "跨Agent协同"]
author: my_agent
---

【经过】需要让 my_agent 在执行任务时主动驱动本机已登录的 WorkBuddy（CodeBuddy 桌面端）干活。
它的 REST 网关要开机随机密码（拿不到），但 agent runtime 另开了一套 ACP：127.0.0.1:56737 的
/api/v1/acp，对回环地址免鉴权。握手顺序：POST /api/v1/acp/connect（空 body）拿
connectionId + sessionToken，之后每个 JSON-RPC 带 acp-connection-id / acp-session-token 两个头，
依次 initialize → session/new(cwd, mcpServers) → session/prompt(sessionId, [{type:text,text}])。
把它封装成 tools/local/workbuddy.py（本地工具目录会被自动发现且 schema 常驻），
这样模型每轮都能看见它、可以自主调用；再配一个 skills/ 里的用法说明（triggers 含 workbuddy），
关键词命中时正文注入。

【关键教训】
1. 响应是 SSE + 通知混合流：正文在 session/update 通知的 agent_message_chunk.content.text，
   要按顺序、无分隔符拼接；agent_thought_chunk 全是噪音要丢；最终 result 在 id 匹配的那条里。
2. session/new 会复用同一 cwd 下的已有会话，历史事件随流回传——直接拼所有 message_chunk
   会把旧回答混进来（实测出现"收到收到收到收到done"）。修法：按事件 _meta 里的
   conversationRequestId 分组，只取与最终 result 同组的正文与工具调用；没有最终 result
   才退化成"取最后一组"。注意：本轮没有工具调用时，别再退回上一组的工具调用。
3. 自己进程的工具超时（这里 300s）会先掐断等待：长任务用 start（后台线程跑 prompt，
   模块级 dict 按 sessionId 存中间状态）+ poll 取回，不要在一个 ask 里死等。
4. 错误路径要能自证：端口不通给 ConnectionRefusedError 的友好提示；poll 找不到 session
   要说明"换进程取不回"。

【可复用步骤】
① 先探活（connect + initialize）确认通道在，再谈能力；
② 把协议细节封装成工具，风险等级 high、approval=auto（否则 never 策略下会被静默拒绝）；
③ 用一次真实多步任务（让它建文件并读回）做端到端验证，不要只看握手成功；
④ 确认工具进了常驻 schema 列表（list_openai_schemas）且技能能匹配（SkillManager.match）；
⑤ 临时探针脚本用完删掉，只留工具与技能说明。
