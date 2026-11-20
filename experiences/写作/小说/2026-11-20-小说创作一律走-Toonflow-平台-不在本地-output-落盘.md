---
title: 小说创作一律走 Toonflow 平台，不在本地 output 落盘
domain: 写作/小说
date: 2026-11-20
tags: ["Toonflow", "写手平台", "小说", "题材偏好", "本地清理", "MCP"]
author: my_agent
---

【经过】用户已接入 Toonflow 写手平台（MCP，工作区 /opt/toonflow/data/workspaces，含 document 文档面板），并明确要求：以后写小说只能在平台上写，本地 output 不再落小说正文；同时纠正了"总写外卖题材"的偏差（用户要的是《我挨骂就变强》而非外卖员那本）。

【教训】1) 不要自作主张反复回到同一题材（外卖规则），用户纠正过就要换轨；2) 小说正文不要写进本地 output/novel 等目录，应直接走 Toonflow 文档/工作区；3) 删除本地小说属不可逆操作，且 output 根目录混着技术报告(非小说)、清理记录、测试报告等，必须先按"是否小说正文"分类，只删小说文本，保留媒体产物与非小说文档。

【可复用步骤】写小说任务：用 Toonflow（getAppState 定位工作区 → openProject/切 document 面板 → writeDocument 写入并保存），产物留在平台工作区，不在本地 output 落 .md。清理本地小说时，排除 articles/、writing_notes/、技术报告类 .md。
