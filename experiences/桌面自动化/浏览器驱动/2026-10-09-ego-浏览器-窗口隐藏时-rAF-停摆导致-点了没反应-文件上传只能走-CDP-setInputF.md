---
title: ego 浏览器：窗口隐藏时 rAF 停摆导致"点了没反应"；文件上传只能走 CDP setInputFiles
domain: 桌面自动化/浏览器驱动
date: 2026-10-09
tags: ["ego浏览器", "自动化点击无效", "rAF停摆", "ElementUI", "文件上传", "setInputFiles"]
author: my_agent
---

【经过】
用 ego 后端浏览器在 DSP TMS 上做设备增删的界面实操。前几步点击都正常，中途出现"点了完全没反应"：点 Element Plus 的「下一步」「删除」都返回 clicked 但页面无变化。

【失败点 / 关键教训】
1) 根因是**页面不可见**：`document.visibilityState === 'hidden'` 时 Chromium 不再产生帧，rAF 不触发 → CSS transition 永不结束 → Element Plus 的下拉/菜单/弹窗停在 enter/leave 的初始态，高度恒为 0；此时元素虽在 DOM 里，点击落在 0×0 上等于点空气。
   判定方法：`js` 里跑一个 rAF 计数 Promise（超时/只有 0 帧即为停摆），或直接读 `document.visibilityState`。
   修复：`node <dsh-ego-browser>/runtime/ego-linux/bin/ego-browser.mjs --open`（配合 EGO_LINUX_CHROME）把 agent 窗口显示出来，帧立刻恢复（实测 400ms 25 帧）。
2) 动画卡住还有个残留形态：弹窗关闭后 `el-overlay` 停在 `dialog-fade-leave-active` 且 display 仍为 block，看不见但挡点击——刷新页面才干净。
3) 同一界面里常有**同名元素藏在隐藏分支**（两个「下一步」，一个 0×0）。按 innerText 找按钮时必须再过滤 `getBoundingClientRect().width > 0`，否则点到隐藏那个。
4) 文件上传：`fill` 对 `input[type=file]` 无效（只是往框里打字）。ego 运行时支持 CDP `DOM.setFileInputFiles`，脚本里是 `await page.locator(sel).setInputFiles(path)`；后续已把它接成 browser 工具的 `upload` 命令。
5) 导入类功能别信"提交成功"的提示就完事：要看它给的失败清单（常见 `sn exist`）并回列表核对条数变化。

【可复用步骤】
- 点击无效先自检三样：`document.visibilityState`、rAF 是否在跑、目标元素 rect 是否 >0。
- 窗口可见性用 CLI `--open` 恢复，不要重启浏览器（会丢登录态与已打开页面）。
- 定位元素优先用"可见 + 文案"双条件；需要填文件就 setInputFiles。
- 结果核对以列表/接口返回的真实数据为准，不看 toast。
