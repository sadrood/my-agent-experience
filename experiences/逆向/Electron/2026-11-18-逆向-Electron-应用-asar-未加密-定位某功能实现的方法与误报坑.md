---
title: 逆向 Electron 应用（asar 未加密）定位某功能实现的方法与误报坑
domain: 逆向/Electron
date: 2026-11-18
tags: ["asar", "electron", "koffi", "ffi", "逆向", "false-positive", "highlight.js", "native-addon", "uiohook", "桌面自动化"]
author: my_agent
---

【经过】目标 Electron 应用装在 D:\workbuddy，WorkBuddy.exe 204MB 内嵌运行时，resources/app.asar 172MB。

1. 解析 asar 无需 asar 包/网络：header 是 Chromium Pickle，绝对偏移 16 处先读一个 uint32 得到 JSON 长度，直接 struct.unpack('<IIII', f.read(16))[3] 即 header 长度，读出来 json.loads 就是完整文件树（含每个文件的 size/offset）。本次拿到 12293 个文件。把 header dump 成 json 后可反复离线分析，不必解包 172MB。
2. 定位某能力用"全文二进制搜 API 名"最快：把整个 asar 读进内存，regex 搜 GetWindowRect/EnumWindows/SendInput/BitBlt/desktopCapturer/UIAutomation 等 Win32 符号，再用 header 的 offset 表 bisect 反查命中属于哪个文件。比逐个解包 grep 快得多。

【失败点/关键教训】
- 最大坑：搜到 SendInput 差点当成"用 Win32 注入输入"的铁证。看上下文才发现是 Highlight.js 的 AutoHotkey 高亮词表（一整串命令名枚举）。字符串命中≠实现调用。必须打印命中点前后 1KB 看语境：真实现一般有 dll.open / func / proc 之类绑定语句伴随，纯标识符列表就是词表/文档/类型声明。
- 同理，asar 里搜到大量 koffi 命中，实为 koffi 自身 bundle（libffi 内部字符串），不代表业务代码在用 FFI；user32 命中是 "user32UserDefinedTransaction" 这类无关权限串。
- 依赖名容易误判：@ban-dal/os-bridge 听着像"操作系统桥接=电脑操控"，读 package.json 才发现描述是 "Typed native notification diagnostics"（通知诊断）。凡是可疑 native 包，先看 package.json 的 description 再下结论。
- 用户数据目录跨用户不可达：当前进程环境是 C:\Users\Administrator，应用数据在另一用户（15104）名下，python 遍历 C:\Users 看不到、直接访问报 FileNotFoundError。逆向按需安装的组件前要先确认"进程实际以哪个用户跑"（Get-Process 拿 Path 只给安装路径，不给数据目录）。
- PowerShell 里 `"$env:APPDATA\WorkBuddy\Users\"` 结尾反斜杠会转义右引号导致解析失败返回码 1；且对可能巨大的目录（含 vm/ 虚拟磁盘）加 -Recurse 会超时。改用 python + pathlib 显式路径、限层遍历。

【可复用步骤】
1. struct 解 asar header → dump 文件清单 json（size/offset）
2. 看 package.json main 与顶层目录名（main/preload/renderer/cli/native）判断分层
3. 全文 regex 搜目标能力的关键 API/符号 → bisect 反查所属文件
4. 每个命中打印 ±1KB 上下文，排除词表/文档/自身 bundle
5. 可疑 native 包读 package.json description 定性
6. 若关键 API 全为 0 命中 → 结论是"该能力不在主包内"，转去查按需安装位置（AppData connectors/plugins/cache 类目录）与外部进程
