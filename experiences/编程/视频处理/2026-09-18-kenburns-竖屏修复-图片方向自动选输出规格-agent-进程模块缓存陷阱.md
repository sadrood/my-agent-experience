---
title: kenburns 竖屏修复：图片方向自动选输出规格 + agent 进程模块缓存陷阱
domain: 编程/视频处理
date: 2026-09-18
tags: ["kenburns", "竖屏", "video_edit", "模块缓存", "zoompan", "漫剧"]
author: my_agent
---

【经过】用 Toonflow 出好的 736x1312 竖屏分镜图走 kenburns 路线做漫剧，产出却是 1280x720 横屏、画面被压扁。排查发现 models/video_edit.py 的 kenburns 用 zoompan 且 s 写死 self.width x self.height（默认 1280x720），竖图被强转横屏。修复：kenburns 开头用 self.probe(image) 读图宽高，竖图(ih>iw)且默认配置为横屏(w>h)时交换 w/h 得竖屏规格；探测失败回退默认，测试假图(64字节)不受影响。【关键教训】① 写交换条件别写反：判断"配置为横屏"应是 w>h，不是 h>=w。② 改完代码后**当前 agent 进程内 tools 层调用仍走旧模块缓存**（sys.modules 缓存），表现是"代码改对了但工具调用还是旧行为"——验证新逻辑要用新起的 python 进程（terminal 跑 .venv\Scripts\python -c），不要用 agent 内工具复测，否则会误判修复失败。③ kenburns 输出规格统一(1280x720 或 720x1280)保证 concat 无损，字幕工具会自动按高度 3.2% 换算字号（竖屏 41px）。【可复用步骤】1) 定位 zoompan s 参数来源 2) 加 probe 探测+方向判断 3) 跑 test_video_edit.py 确认断言 s=1280x720 仍过 4) 新进程验证竖屏输出 5) 全量 pytest。
