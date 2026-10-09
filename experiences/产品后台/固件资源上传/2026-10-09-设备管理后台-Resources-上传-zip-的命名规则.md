---
title: 设备管理后台 Resources 上传 zip 的命名规则
domain: 产品后台/固件资源上传
date: 2026-10-09
tags: ["命名规则", "固件上传", "zip", "Resources", "CUST", "CERT"]
author: my_agent
---

【场景】在设备管理后台（Device Mgmt → Resources）上传固件/证书资源包时，前端会按文件名做校验，不符合就报 "文件名不符合规则 / The file name does not comply with the rules"。

【规则】文件名格式：
`<CUST|CERT>_<机型>-<两位码>_<客户名>_<版本号>.zip`

- 第一段只能是 CUST 或 CERT（全大写）：CUST = 定制包，CERT = 证书包，决定资源类型
- 第二段是目标机型 + 连字符 + 两位码，如 D30M-MU
- 第三段客户名，纯字母，如 ALIAPAY、PAGGO
- 第四段版本号，形如 V0.0.1
- 例：CUST_D30M-MU_ALIAPAY_V0.0.1.zip

【上传后资源名】列表里显示的资源名 = 文件名去掉「机型-两位码」段和「版本号」段再拼回机型，
例：CUST_D30M-MU_ALIAPAY_V0.0.1.zip → 资源名 CUST_D30M_ALIAPAY（版本另列 V0.0.1）。

【可复用步骤】
1. 规则不在页面上，在前端校验代码里：扒 Resource 页组件源码（js）比翻文档快。
2. 拿到线上已有成功条目（名称/版本/来源 Local File）反推格式，最准。
3. 源文件名字被揉过时（如 CERT_PAGGO_D30M-MU_GEN.zip），用包内 cert.ini 的 Version / customerID 与时间戳确认它是哪个客户哪一版，再重命名（复制、不改原文件）。
