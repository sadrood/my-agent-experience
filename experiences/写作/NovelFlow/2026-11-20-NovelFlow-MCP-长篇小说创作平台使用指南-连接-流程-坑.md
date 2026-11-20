---
title: NovelFlow MCP 长篇小说创作平台使用指南（连接、流程、坑）
domain: 写作/NovelFlow
date: 2026-11-20
tags: ["novelflow", "MCP", "小说创作", "AI写作", "异步任务", "工具使用"]
author: my_agent
---

【是什么】NovelFlow：部署在服务器上的 AI 长篇小说创作平台，经 SSH MCP 接入（本机 ~/.my_agent/mcp_servers.json 已有 novelflow 条目，SSH 免密已通）。流水线：架构师出大纲 → 写手起草 → 多模型七维审校 → 主编裁定 → 修订循环 → 定稿并更新角色/伏笔记忆。

【连接与装载】工具名形如 mcp_novelflow_*；按需装载：先 tools list 找到，再 tools load mcp_novelflow_xxx，下一轮才能调用。首次连接先调 get_agent_guide，可拿回官方指南全文（SKILL.md + 33 个工具参数表），不用凭记忆猜参数。服务器源文档：/opt/novelflow/skills/novelflow/SKILL.md 与 references/tools.md；本地已备份到 tools/local/novelflow_skill/。

【ID 规则（最易错）】project_id / chapter_id / run_id 都是数据库自增 id，不是章号；章号是 idx。写"第 5 章"：先 list_chapters(project_id) 取 idx==5 那条的 id，再 generate_chapter(chapter_id=...)。get_project 返回 {"project":{...}}，list_chapters 返回 {"chapters":[...]}，取字段前先看外层 key。

【新书流程】create_project(title,genre,style,premise) → generate_outline(project_id,chapter_count,direction) → 轮询 get_run → list_chapters 确认章节已播种 → generate_chapter(第一章 chapter_id) → 轮询 → get_chapter 读正文。大纲会同时播种角色/世界观/伏笔，同一项目不要重复生成大纲（章节是 INSERT OR IGNORE，但角色/世界观/伏笔直接插入，会重复）。

【异步轮询】所有生成动作都是异步：调用立即返回 run_id，每 5–10 秒 get_run(run_id) 一次，直到 status 不是 running（done/error/cancelled）。run.log 最后一条是当前阶段（architect/writer/editor/rules/arbiter/memory/batch）。status=error 时读 run.error 里的异常与 traceback 尾部，先判断原因再重试，不要盲目重跑烧额度。用户要停用 cancel_run。

【分步控制】draft_chapter 仅起草；review_chapter 仅审校；revise_chapter(chapter_id, must_fix=[...]) 按意见修订；accept_chapter 定稿（写摘要、更新角色状态与伏笔台账）。generate_chapter 跑完整流水线，结束状态是 reviewed（已审未定稿），不会自动定稿，除非 config.auto_accept=true。

【批量】batch_generate(project_id, from_idx, to_idx) 逐章跑完整流水线，单次最多 20 章；默认 batch_continue_on_error=true，某章失败记日志跳过继续。

【模型/预算配置】生成用的模型由 NovelFlow 决定，不是 agent 自己的模型。改法：get_project 拿完整 config → 改字段 → update_project 整体回传。config 是整体替换不是深合并，只传一个字段会把其他配置冲掉。常用字段：writer_model（写手/架构师）、editor_models（审校数组）、arbiter_model（主编）、summary_model、writer_fallback_models、target_words（默认3000）、max_rounds（默认2）、accept_threshold（默认7.5）、auto_accept、writer_max_tokens（默认12000）、context_token_budget（默认24000）。模型名必须来自 list_models。改配置只影响新任务。

【错误处置】429 all_channels_saturated = 上游渠道并发满（不是模型下线），等 1–3 分钟或减少 editor_models，不要连发；返回空内容 = 推理模型把 max_tokens 花在 thinking 上，把 writer_max_tokens 提到 16000 或换非推理模型（客户端会自动翻倍重试一次）；输入超上下文窗口 = 调小 target_words / context_summary_count 或换大窗口模型；所有编辑模型审校失败 = 换/减 editor_models 重跑 review_chapter。

【边界】不直接改数据库，写操作全走 MCP 工具；删除项目/章节不可恢复，执行前必须向用户确认；批量单次≤20 章；不要把 agent 模型配置当成小说模型配置；MCP 经 SSH 注入 token，不要在输出/日志/文件里回显凭据。
