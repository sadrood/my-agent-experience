---
title: Electron 桌面 Agent（WorkBuddy）外部控制面：先分清"哪个主体在干活"再谈调用
domain: 安全/逆向工程
date: 2026-10-12
tags: ["electron", "asar", "workbuddy", "命名管道", "IPC", "逆向", "CDP"]
author: my_agent
---

【经过】
用户要"直接调用 WorkBuddy 桌面端干活"，我上一轮误用了它自带的 agent CLI（app.asar.unpacked/cli/bin/codebuddy，包名 @genie/agent-cli），该 CLI 与桌面端账号体系独立，必须自己 /login，于是全部失败。

【关键教训】
1. Electron 桌面包里往往同时存在"壳"和"CLI"两条入口，CLI 打包名与产品名不一致（产品 WorkBuddy / CLI 叫 codebuddy / 包名 @genie/agent-cli）。用户说的"用 WorkBuddy"指的是已登录的桌面端，不是那条 CLI。动手前先确认"登录态在谁身上"。
2. 桌面端的真实控制面是 Electron IPC 桥（preload 暴露的 window.__wbInvoke / wb:event，命令名形如 cloudAgent:createConversation、automation:*），只有进程内 renderer 能调；外部只能走设计好的外部通道。
3. 外部通道写法（WB 实测）：~/.workbuddy/wbipc/endpoint.json 给出命名管道 + ticket；连上后 newline-JSON + HMAC-SHA256 握手（ticket_id=sha256(ticket)[:16]；proof=HMAC(ticket, 长度前缀拼接[role,protocol,endpoint,clientNonce,serverNonce])），ready 后 JSON-RPC：broker/ListPipes、broker/GetPipe{pipe}、c:<pipe>/<method>。WB 只暴露 3 条管道：wb.request(http.fetch，会自动注入登录 Bearer → 等于借壳访问其后端)、wb.entitlement、wb.wecom-a2a；**没有**"让 agent 干活"的管道（设计上不开放）。
4. 排查要点：本地固定端口可能只是探活（WB 的 18488 = /workbuddy/probe 只读）；自动化任务是云端调度（/v2/as/scheduler/tasks）+本地镜像，直接往 SQLite 插行不会触发执行；深链 workbuddy://chat/<sessionId> 只能打开既有会话，带不了 prompt。
5. 真正可用的"让桌面端干活"路径：① 带 --remote-debugging-port 重启后 CDP 驱动 renderer（需要用户放行一次重启）；② IM 通道（该机已绑定 QQ 机器人，agent-im-bindings.json 里 status=bound），直接给机器人发消息即用其登录态执行。

【可复用步骤】
- asar 只读解析：头 8 字节 + hsize → JSON 目录树；offset 相对数据区起点 (8+hsize)。按字节模式全文件搜索再把偏移映射回文件路径，能快速定位"哪个文件注册了什么"，比逐个解包快得多。
- 版本漂移：先核对磁盘上 app.asar 内文件大小与旧解包副本是否一致（WB 5.7.7→5.8.1 就变了），别拿旧副本下结论。
