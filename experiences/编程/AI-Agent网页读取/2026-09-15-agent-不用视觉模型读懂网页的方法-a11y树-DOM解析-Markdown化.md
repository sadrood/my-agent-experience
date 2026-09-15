---
title: agent 不用视觉模型读懂网页的方法（a11y树/DOM解析/Markdown化）
domain: 编程/AI-Agent网页读取
date: 2026-09-15
tags: ["web-agent", "accessibility-tree", "aria-snapshot", "DOM解析", "jina-reader", "网页读取"]
author: my_agent
---

【经过】学习“如何让 agent 不靠截图+视觉模型读懂网页”，查阅 MDN 无障碍树、Playwright aria snapshot 文档、CDP Accessibility 域、WebDriver BiDi、Jina Reader，并对照自家 tools/browser.py 的 snapshot/text 实现。

【关键结论】三层方法，按推荐度排序：
1. 无障碍树（推荐首选）：浏览器由 DOM 计算生成 a11y 树，节点含 role/name/description/state，本就是给屏幕阅读器的文本化描述。
   - Playwright ≥1.49: locator.aria_snapshot(depth=N) 得到 YAML（role "name" [checked/disabled/level=..]）；旧版回退 page.accessibility.snapshot() 字典树。
   - CDP: Accessibility.getFullAXTree/getPartialAXTree/queryAXTree；WebDriver BiDi: browsingContext.locateNodes 支持 role+accessible name 语义定位。
   - 自家 tools/browser.py 的 _accessibility_snapshot() 已是该做法的参考实现。
2. DOM 文本/结构提取：page.inner_text("body") 拿可读文本；BeautifulSoup/lxml 去掉 script/style/nav/footer 后取 main/article 正文；正文提取库 readability / trafilatura / html2text。
3. 一键 Markdown：URL 前加 r.jina.ai/（Jina Reader 免费）返回带 Title/URL Source/发布时间 的干净 markdown，最省事。

【为什么优于视觉】：截图贵/慢/token 大/会幻觉、拿不到精确文本和属性、无法绑定可操作元素；DOM/a11y 方式便宜精确且可直接生成 role+name 选择器来点击输入。视觉模型只留给 canvas/图片为主或需要看渲染效果的页面。

【可复用步骤】：1) 优先尝试 aria_snapshot/locator 语义查询；2) 设 max_depth/max_nodes 截断超大快照；3) a11y 为空（canvas）再回退 text/html/截图；4) 抓正文时剔除 script/style/nav/footer。
