---
title: 调研 GitHub 仓库并提炼可借鉴点（以 claude-code 官方仓为例）
domain: 研究/开源仓库调研
date: 2026-10-13
tags: ["github调研", "claude-code", "hook协议", "输出折叠", "tar.gz下载"]
author: my_agent
---

【经过】
要调研 https://github.com/anthropics/claude-code 有什么可借鉴。先核实仓库性质：它不是 CLI 源码仓（本体闭源分发），真正有料的是 plugins/（13 个官方插件）、mods/（4 个内置插件 TS 源码：agents-md、sec-default、diff、telemetry）、plugin-dev 的 7 份规范 SKILL.md、examples/、.devcontainer。
取清单用 GitHub API（api.github.com/repos/OWNER/REPO/git/trees/main?recursive=1）；取正文用 codeload.github.com 的 tar.gz（12MB）下载后 tarfile 解压到 output/ 子目录，本地逐文件精读。

【失败点/教训】
1) git clone github 仓库在本机网络下极慢（一分钟无进展），改用 codeload tar.gz 一次拿全仓快得多；python 里 tarfile 可用（shutil 被黑名单禁用），extractall 用 filter=data 兼容新版本。
2) 浏览器读长页面：js 取 innerText 超过约 2-3KB 就触发输出折叠，随后 context recall 也被折叠，形成死循环。正确做法是 js 分片返回（每次 <=1400 字符），或下载到本地用 python 分片打印；python 一次性 print 大段同样会折叠，也要切片。
3) 报告里的数字（插件个数等）必须数一遍再写，实测官方插件是 13 个而非 14 个。
4) grep 上下文式匹配（前后各 40 字符）在同行长文本不足时会匹配不到，别据此断定内容不存在。

【可复用步骤】
1. 先判断仓库是源码仓还是插件/示例/issue 仓，决定读什么。
2. 清单走 API trees，正文走 codeload tar.gz + tarfile 解压。
3. 只精读有机制价值的文件：hooks 注册 json、核心引擎 py、规范 SKILL.md、README 里讲设计取舍的段落。
4. 提炼时做三段式映射：外部做法 → 本仓现状（具体文件）→ 落地建议；写建议前先读本仓对应模块，避免建议已实现的功能。
5. 收尾删除中间产物（tar.gz、解压目录），只留分析报告。

【Claude Code 侧可复用要点】
- hook 协议：事件 PreToolUse/PostToolUse/Stop/SubagentStop/SessionStart/UserPromptSubmit/PreCompact；stdin 收 JSON；退出码 0 通过、2 阻断并把 stderr 回喂模型、1 只给用户；stdout JSON 决策（permissionDecision allow|deny|ask、updatedInput；Stop 用 decision approve|block）；asyncRewake 后台审完回灌消息。
- hookify：markdown + frontmatter 规则（event/action warn|block/conditions/field/operator），即时生效，异常一律 fail-open。
- security-guidance：25 条正则只做提示不做拦截（教育性文案）；用户自定义正则要过 ReDoS 检测；策略文件分层且 8KB 预算从尾部截断。
- code-review：两阶段（并行找问题 → 独立子代理逐条验证）+ 显式假阳性清单；模型分档 haiku 预筛、sonnet 合规、opus 深审。
- mods/sec-default：组织策略高于用户插件，策略读取 fail-closed；agents-md：CLAUDE.md 分层加载 + local 覆盖 + 子目录按需附着。
