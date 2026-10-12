---
title: WorkBuddy wbipc 外部 IPC 通道：握手协议与打通步骤
domain: 逆向工程/桌面应用
date: 2026-10-12
tags: ["workbuddy", "ipc", "命名管道", "HMAC握手", "逆向工程", "A2A"]
author: my_agent
---

【背景】WorkBuddy（腾讯 Electron 桌面 AI Agent）5.8.1 新增一条面向外部程序的 IPC 通道 wbipc，运行时信息在 ~/.workbuddy/wbipc/endpoint.json（{endpoint:\\.\pipe\wbipc-<hash>, ticket:<随机串>}）。

【失败点】自研客户端只连管道、被动等 session_challenge，一直超时被踢。看主进程 server.js 才发现服务端 state='hello' 时**只接受 session_hello**，客户端不发就永远等不到 challenge（握手 15s 超时 destroy，node 因无 handle 静默 exit 0，还容易误判成"协议变了"）。

【正确握手四步】
1) 连上命名管道后立刻发 {type:'session_hello', protocol_min:1, protocol_max:1, client_nonce:<base64url>, ticket_id: sha256(ticket).hex.slice(0,16), client:{kind,id,version}}；
2) 收 {type:'session_challenge', server_nonce, server_proof}；
3) 校验 server_proof = HMAC-SHA256(key=ticket, msg='wbipc-s'|protocol|endpoint|clientNonce|serverNonce 各自 4 字节大端长度前缀拼接)，然后回 {type:'session_prove', client_proof:HMAC(...,'wbipc-c'|...)}；
4) 收 {type:'session_hello_ack', connection_epoch, pipes:[...]}。

【调用方式】帧是 \n 分隔的 JSON（上限 1MB）。先 {jsonrpc:'2.0',id,method:'broker/GetPipe',params:{pipe}} 换到 channel 名 'c:<pipe>'，再以 method='c:<pipe>/<method>'、mode='call'|'stream'（必须与方法的 kind 一致）调用。协议常量在 asar 的 main/server.js 里，搜 'wbipc' / 'session_hello' 即可定位。

【实测结果（5.8.1）】对外可见 3 个 pipe：wb.entitlement/subscription.check → {allowed:true}；wb.wecom-a2a/authorize → {allowed:true}；wb.request/http.fetch → 主进程自动注入用户 Bearer token 后代理请求后端，GET / 返 200（UI HTML）。http.fetch 支持 POST/PUT/PATCH/DELETE，但整条通道**没有发消息/触发会话的方法**；wecom-a2a 背后是 A2A 协议（常量含 /a2a/cards/self、/a2a/credentials、/a2a/upstream/），是官方给外部 agent 的对接路径。

【可复用提醒】(a) 自研协议客户端卡在"服务端不主动说话"时，先怀疑客户端漏发首帧；(b) node 里 process.exit 会截断异步 stdout，探针结果用 fs.writeFileSync 落盘更可靠；(c) 长输出在受限终端会被折叠，分段打印每次 <1.5KB；(d) 通过桌面端已登录凭证代理请求属敏感操作，继续探测后端业务路径会被安全策略拦，别硬绕。
