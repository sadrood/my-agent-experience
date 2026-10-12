---
title: 把"扫码绑定 QQ 机器人"内置进自己的 agent（腾讯轻量绑定接口，Python 复刻 node 包）
domain: Agent工程/工具集成
date: 2026-10-12
tags: ["QQ机器人", "扫码绑定", "腾讯轻量绑定", "逆向复用", "工具内置", "qqbot"]
author: my_agent
---

【经过】目标：让 my_agent 具备 WorkBuddy 那样的"手机 QQ 扫码就连上机器人"能力，不必去开放平台手动建应用、复制 AppID/AppSecret。WorkBuddy 桌面端的连接层走 npm 包 `@tencent-connect/qqbot-connector`（app-server 里 QQBindService.startBind 调它的 startQrConnect），把包解出来读 dist/esm/*.js（实现只有 ~2KB），逻辑全在里面，于是用 Python 复刻成 agent/qqbot_bind.py，并接到 `python -m agent.qqbot --bind`。

【关键事实（腾讯轻量绑定接口，全部无需预置凭据）】
1. `POST https://q.qq.com/lite/create_bind_task` body `{"key": base64(32字节随机)}` → `{retcode:0, data:{task_id}}`
2. 二维码 URL：`https://q.qq.com/qqbot/openclaw/connect.html?task_id=<id>&source=<标识>&_wv=2`（二维码是手机扫的，host 固定 production）
3. 轮询 `POST https://q.qq.com/lite/poll_bind_result` body `{"task_id": id}` → `data.status` 0未知/1等待/2完成/3过期；完成时给 `bot_appid` + `bot_encrypt_secret`
4. 解密钥：AES-256-GCM，密钥=第 1 步那份随机 32 字节，密文 base64 解出后 **前 12 字节 IV、后 16 字节 tag**、中间是密文（Python: `AESGCM(raw).decrypt(iv, body+tag, None)`）
5. WorkBuddy 拿到凭证后还会把频道注册到它自己的云（registerClawChannel）再 enabled；本地桥不需要这层，直接 qq-botpy 连官方 gateway 即可。

【可复用步骤】解包 npm 源码读实现 → Python 复刻（requests 调接口、cryptography 解密、qrcode 出二维码，都惰性 import）→ 写 .env 时"已有真值行就地替换、空占位注释行也填上、别处不动"、密钥不回显不进日志 → 打桩 `_post_json` 做全离线单测（任务/轮询/状态机/过期换码/超时/写 .env）→ 文档与 requirements 的可选依赖注释一起补。

【教训】① 别一上来就撬宿主程序的内部实现：连接层往往就是一个公开 npm 包，包本身自包含、比宿主好读得多。② Guardian 会拦 agent 自己发起的这类对外绑定请求（判"向外部非标准域名发送数据并执行绑定任务"），这是设计使然——绑定本质是交互式的（要人扫码），交给用户跑或让用户口头授权（consent）即可，不要反复重试。③ 二维码既在终端出字符画又落 PNG：Windows cmd 下 print_ascii(invert=True) 输出方块字符没问题，PNG 是兜底。
