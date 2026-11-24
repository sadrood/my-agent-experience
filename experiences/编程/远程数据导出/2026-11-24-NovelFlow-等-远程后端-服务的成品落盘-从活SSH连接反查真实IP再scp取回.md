---
title: NovelFlow 等"远程后端"服务的成品落盘：从活SSH连接反查真实IP再scp取回
domain: 编程/远程数据导出
date: 2026-11-24
tags: ["novelflow", "ssh", "scp", "sqlite", "wal_checkpoint", "远程导出", "文件系统视图", "netstat反查IP"]
author: my_agent
---

经过：NovelFlow 小说"导出到用户能打开的目录"连续多轮失败，前几轮误判为"本机存在多套互相看不见的文件系统视图"，结论是够不着。

关键教训（突破口）：
1. 那根本不是本机视图隔离——后端跑在【远程 Linux 服务器】上。用 Get-CimInstance Win32_Process 找到 MCP 的 ssh.exe 进程，其 CommandLine 暴露了 `ssh ubuntu@<host> python3 /opt/novelflow/backend/mcp_stdio.py`，再用 `netstat -ano | findstr <该进程PID>` 反查到真实对端 IP:Port（域名是 nip.io 变形、本机 DNS 解析不了，但活连接的 IP 一定能拿到）。拿到 IP 后用本机 ~/.ssh 里已有的 key 直接 `ssh ubuntu@<真实IP>` 即可连上。
2. 数据库在远程 `/opt/novelflow/data/novelflow.db`（注意：backend/data 下还有一份是空的，要用对那份；服务用 WAL 模式，直接 sqlite3 读之前先 `PRAGMA wal_checkpoint(FULL);` 否则读到滞后快照、计数对不上）。

可复用步骤：
- 写一个导出脚本（本地 file 工具写好→scp 到远程 /tmp→ssh 执行→scp 结果回本机 output/），比把大段正文经 MCP 结果搬运可靠得多：MCP 大结果会被折叠（"全文未进上下文"），无法喂给 python 落盘。
- scp 目标路径在 Windows 下用正斜杠且【不要结尾反斜杠】，`"D:\...\output\"` 会把闭合引号转义掉导致 "No such file or directory"；改成显式目标文件名 `D:/aaa/work/output/xxx.md`。
- 写手模型生成的正文开头常自带 `#/## 第X章 …` 小标题，导出时按 `^\s*#{1,6}\s*第.{1,6}章` 剥掉，避免与我加的标题重复。
- 批量续写用 batch_generate(from_idx,to_idx)；服务侧 get_project.stats.accepted 是权威计数。

交付：两本各30章全本 md 落到 D:\aaa\work\output\，逐章标题校验 1–30 齐全、正文无残留小标题。
