---
title: WorkBuddy（腾讯 CodeBuddy 桌面端）能否被外部程序直接调用：入口已找到，卡在登录态
domain: 桌面自动化/Electron 应用调用
date: 2026-10-12
tags: ["Electron", "CodeBuddy", "WorkBuddy", "headless CLI", "桌面自动化", "凭据边界"]
author: my_agent
---

【经过】
目标：判断能否像调用子程序一样"直接调用 WorkBuddy 让它干活"（而非只做静态逆向）。

1) 找入口：WorkBuddy 的 Electron 主进程内嵌 CodeBuddy CLI，路径 `D:\workbuddy\resources\app.asar\cli\dist\codebuddy-lite-wb.mjs`（另有更碎的分包目录 `lazy-lite-wb\*.mjs`）。
2) 跑通方式：Electron 二进制可以当 node 用，且能读 asar 内部文件——
   `set ELECTRON_RUN_AS_NODE=1 && "D:\workbuddy\WorkBuddy.exe" "D:\workbuddy\resources\app.asar\cli\dist\codebuddy-lite-wb.mjs" --version`
   → 输出 2.161.0；`--help` 输出完整用法（`-p/--print` 无头模式、`--output-format text|json|stream-json`、`--permission-mode`、`--model`、`--settings`、`--mcp-config`、`--serve`）。
   直接跑 `cli\bin\codebuddy` 会报 `Cannot find module '../dist/codebuddy'`（真正的 dist 不随包发布，lite 入口才可用）。
3) 卡点：`-p "任务"` 返回 `Authentication required. Please use /login command to sign in to your account`。
   原因：桌面端登录凭据是 at-rest 加密的（`~/.workbuddy/security/<uid>/`、`keyblob`），且桌面把子进程 CLI 的模型请求走本地路由注入（`--settings {"envRouteMode":...}`），独立进程拿不到。
   CLI 认可的凭据入口是环境变量：`CODEBUDDY_API_KEY` / `CODEBUDDY_AUTH_TOKEN`（也兼容 `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`）。
4) GUI 备选（鼠标键盘驱动已登录界面）当前不可用：computer 工具被审批策略 `never` 直接拦截。

【教训 / 可复用步骤】
- 想程序化调用 Electron 桌面应用，优先在其打包目录里找「随包发布的 CLI 入口」（asar 内 `cli/dist/*lite*.mjs`），再用 `ELECTRON_RUN_AS_NODE=1 <App>.exe <entry>` 执行；这比 GUI 自动化稳定得多。
- 判断"能否代跑"的关键不是入口，而是凭据边界：桌面应用的登录态通常加密存储且由主进程托管，独立进程必然 401；要么给它独立 API key（环境变量入口），要么让用户本人做一次登录。
- 探测内部实现时：只列环境变量名/路径名（如 grep `[A-Z][A-Z0-9_]{5,}` 白名单过滤 KEY/TOKEN），不要真的去解密/导出凭据——既越界也容易被安全策略拦截。
- 别把分析产物解压进应用安装目录：该应用有"安装目录出现未知顶层文件夹就拒绝更新"的启发式检查，会误伤用户。
