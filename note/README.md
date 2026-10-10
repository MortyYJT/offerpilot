# note/ 目录说明

`note/` 放中文的规划、决策和学习笔记。英文的工程文档在 `docs/`，当前有效的规格在 `openspec/specs/`。

## 现在有效（会持续更新）

| 文件 | 用途 |
| --- | --- |
| `handoff.md` | 会话交接，**唯一的当前状态文件**。每次会话结束时整篇重写；读之前先用 `git log` 和 `gh pr list` 核对 |
| `decisions.md` | 用户的每个决定及理由。只追加，不改旧条目 |
| `product/vision.md` | 产品愿景和定位 |
| `product/roadmap.md` | 已排序的路线图。每一条开工前都要先变成 OpenSpec change |
| `product/backlog.md` | 想法池，还没排序的想法放这里 |
| `interview/` | dsh 写的阶段复盘和面试题，见 `interview/README.md` |

## 历史记录（只读，不再更新）

下面这些是引入 OpenSpec 之前（阶段一到 M3）留下的过程文件。源码注释和 OpenSpec 归档里还会引用它们，所以留在原位不搬走。新的设计请写进 OpenSpec change，不要再往这些目录里加文件。

| 文件 | 内容 |
| --- | --- |
| `superpowers/specs/` | 阶段一、阶段二数据库、M2 的设计文档 |
| `superpowers/plans/` | 阶段一到 M3a 的实施计划 |
| `stage-1-MVP.md` | 阶段一到 M1 的开发过程日志 |
| `verification/stage-1-skeleton.md` | 阶段一验收证据的中文原件，英文版在 `docs/verification/` |
| `testing-m3.md` | M3 合并当天写的测试指引。里面的端口和服务状态已经过期 |
