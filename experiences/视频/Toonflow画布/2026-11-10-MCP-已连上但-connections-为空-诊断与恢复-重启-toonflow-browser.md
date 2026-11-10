---
title: MCP 已连上但 connections 为空：诊断与恢复（重启 toonflow-browser）
domain: 视频/Toonflow画布
date: 2026-11-10
tags: ["toonflow", "mcp", "connections为空", "toonflow-browser", "systemctl", "故障恢复"]
author: my_agent
---

【经过】用户给 Toonflow MCP 配置（stdio 桥 + https://<host>:18790/mcp + token），要求"连接一下"。mcp_toonflow_getAppState 正常返回，但 connections 为空数组，所有画布工具报"工具已禁用，或所需 Toonflow 页面未连接"。

【诊断路径（可复用）】
1) getAppState 能返回 = MCP 服务器活着，问题在"页面没登记"。别反复重试工具，也别重新 launch 本机浏览器。
2) curl -sk https://host:18790/ → 302 /login；/mcp 裸请求 401 → 服务与 token 校验都在，属正常。curl 直连报错 60 是证书链，一律加 -k。
3) 远端查 headless 页面：curl -s http://127.0.0.1:9222/json/list 看 target 是否还在（http://main.localhost:3100/#/workspace）。本案页面还在、服务 active，但 MCP 注册表仍空 —— 说明页面内 MCP 客户端断链，只能重启页面。
4) 恢复：ssh <user>@<host> 'sudo -n systemctl restart toonflow-browser'，等 15~20 秒再 getAppState → connections 立刻回填（新 connectionId，旧的要重新取）。

【关键教训】
- connectionId 会变，重启后必须先 getAppState 拿新 id，不要沿用旧值。
- 服务名清单：toonflow、toonflow-browser（主实例无头页面）、toonflow-gateway（账号网关，代管 /mcp 与 /login）、toonflow-shuyang(-browser)。18790 由 nginx 反代到 gateway。
- 本机浏览器打开站点会撞证书错误页（NET::ERR_CERT_AUTHORITY_INVALID），而"继续访问"链接点击不稳；其实不需要本机浏览器 —— 连的是远端常驻无头页面，本机登录毫无意义。
- 远端 Ubuntu 用户 ubuntu 免密 SSH 可用（root 不行），sudo -n 免密，可直接 systemctl / journalctl / ss。
- 探测节点能力但不消耗额度：addNode remote-videoGenerationNode → node:getConfig 看 provider/model/mode → 用完 deleteNodes；当前默认 agnes/agnes-video-2.5-flash、mode=text、9:16 720P 5s。
- getConfig/getSettings 结果 4~8KB 就会被工具层折叠，recall 也会再次折叠（拿不到全文）。要精确字段就用 getCanvasNodes 的 fields=["data"]+dataKeys，别指望 recall。
