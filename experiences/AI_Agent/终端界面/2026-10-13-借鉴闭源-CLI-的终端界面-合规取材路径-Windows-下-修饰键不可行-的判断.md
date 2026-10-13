---
title: 借鉴闭源 CLI 的终端界面：合规取材路径 + Windows 下"修饰键不可行"的判断
domain: AI Agent/终端界面
date: 2026-10-13
tags: ["claude-code", "TUI", "终端界面", "pytest-隔离", "合规取材", "Guardian"]
author: my_agent
---

【任务】用户问"claude-code 的终端界面样子不借鉴一下吗"，需要分析并把可借鉴项落地。

【取材路径（关键教训）】
1. 本机装了 claude-code（npm 全局，实为 237MB Bun 打包的 claude.exe）。想从二进制里抠 UI 字符串 → 被 Guardian 当场拦（"未授权的逆向工程"）。**不要重试同类操作**，改走官方渠道。
2. 官方 Interactive mode 文档（docs.claude.com / code.claude.com）信息量足够：键盘快捷键表、模式切换、transcript viewer、task list、/diff 面板、队列消息、prompt suggestions 全都有。用 `urllib.request` 抓 HTML → 去标签 → 存临时文本 → 分段（<1.5KB）打印，绕开工具输出折叠。
3. 工具的 Python 进程会缓存已 import 的模块：改完源码再 import 仍是旧版本（AttributeError）。验证新函数要用 terminal 起新进程（`python -c "..."`）。

【判断结论（可复用）】
- 闭源 CLI 的界面能力大量绑在修饰键上（Shift+Tab/Ctrl+O/Ctrl+T）。cmd.exe 与传统 Windows 终端没有 Kitty 键盘协议，拿不到可靠键序列；"读一行 → 执行"的 REPL 也没有 raw mode。**结论：凡修饰键能力，一律给斜杠命令入口**（如 /diff、/shortcuts），别硬模拟按键。
- 先做"低成本高观感"项：折叠摘要（连续同类工具调用合并成一行）> 全量回看（/transcript）> 一次性面板（/diff、/shortcuts）> 状态行字段。
- 对照时要写清"我方现状取自哪个文件的哪个函数"，否则报告只是复述上游文档。

【顺手发现的既有缺陷（值得记）】
pytest 跨文件污染：tests/test_ui.py 的 fixture 直接 `ui.get_console = lambda...` 赋值且不还原，把模块全局 console 永久指向 StringIO，导致按字母序/指定顺序跑时 tests/test_status_bar.py 的 capsys 用例抓到空输出。
定位手法：**单跑该文件全绿、与另一文件组合就红 = 顺序依赖污染**，用 `pytest A B` 与 `pytest B A` 两种顺序对照即可确认。
修法：fixture 改 `monkeypatch.setattr(ui, "get_console", ...)`（monkeypatch 会自动还原）。

【落地清单】
agent/ui_theme.py 新增 print_git_diff()/print_shortcuts()（一律用 rich.text.Text，diff 内容里的 `[doc](url)` 会被 markup 静默吞掉）；
main.py 加 /diff、/shortcuts 分支 + 补 /help 与 _parse_command 白名单（既有用例 test_every_handled_command_is_parseable 会自动盯一致性）；
docs/COMMANDS.md 同步；补 6 个用例。
