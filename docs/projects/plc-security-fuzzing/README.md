# 基于覆盖率反馈的 PLC 安全性分析

## 项目概览

本项目研究如何使用 Fuzzing 分析 PLC（可编程逻辑控制器）应用程序的安全性。阶段性方案以 OpenPLC 将梯形图等 IEC 61131-3 程序转换为 C 代码，再接入 LibAFL 的覆盖率反馈流程，并探索将外设 I/O 纳入测试的方法。

## 研究路线

1. 将 PLC 程序转换为可在通用环境中执行和插桩的 C 代码。
2. 使用 LibAFL 构建基于覆盖率反馈的模糊测试流程。
3. 对 PLC 的循环扫描、外设输入和竞态等特性建模。
4. 评估漏洞发现能力、覆盖率与测试效率。

## 项目材料

- [开题报告草稿](proposal-notes.md)：背景、PLC 安全问题、相关工作和早期研究思路。文中保留了待完善标记，属于历史草稿。
- [开题答辩演示稿](proposal-slides.md)：研究背景、目标、方案、预期成果与当时的计划。原 PDF 是由该 Markdown 导出的重复成品，且包含个人信息，因此未纳入整理后的仓库。

> 状态说明：上述材料是 2024 年开题阶段的历史记录，其时间计划不代表当前进度。

## 相关知识条目

- [PLC 与工业控制安全文献](../../literature/README.md#plc-与工业控制安全)
- [Para-rehosting 分享](../../presentations/para-rehosting/para-rehosting.md)
- [工业互联网漏洞库与攻击链专利调研](../../topics/industrial-security/patent-landscape.md)
- [LibAFL 论文笔记](<../../literature/LibAFL: A Framework to Build Modular and Reusable Fuzzers.md>)

## 来源整合说明

本目录由 `PPT_private` 仓库的“开题”目录整理而来。原仓库中的 `para-rehosting/para-rehosting.md` 及配图、以及“开题”目录下重复的 `main.md`，已合并到现有的 [Para-rehosting 演示资料](../../presentations/para-rehosting/para-rehosting.md)，不再保留重复副本。
