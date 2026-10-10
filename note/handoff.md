# Session handoff

**This is the only current-state file. Overwrite it at the end of a session; read it at the start.**
Verify it against `git status`, `git log` and `gh pr list` before trusting it.

## Where we are（2026-10-11）

- `main` 仍是 M3 合入后的 `794a911`。三个 PR 等 MortyYJT 合并，互不依赖：
  - PR #13 `claude/workflow-v2`：开发流程 v2 + guard hook；`note/decisions.md`（全部决定及理由，**先读它**）；
    新的愿景 / 路线图 / 想法池；本文件
  - PR #14 `dsh/interview-m1-m3`：M1–M3 中文复盘和题目
  - PR #16 `codex/static-prototype`：静态原型 `/prototype`（关 #15 #17 #18），最新提交 `ea2c91a`
- 产品定位：MortyYJT 留学中介的获客与初筛工具，界面对标指南者，但去掉噪声和广告
- 阶段：**先用静态原型定功能点，再补后端**；M4 已移进想法池

## 原型现状（PR #16）

- 学校库 `/prototype/schools`：海外 346 所（QS 2027 前 300 + 澳洲 40 + 香港 22 + 澳门 10），按 QS 名次排序；
  显示中文名（官方 48 所，常用译名 298 所）、联盟标签（罗素 24 / 澳洲八大 8 / 常春藤 8 / 教资会 8）、QS 名次、
  在线咨询按钮；地区和 QS 区间筛选；校徽 287 所（取自各校官网，其余 59 所用首字圆标）
- 背景测评：本科院校可在教育部全部 3167 所里搜索，自动识别 985 / 211（双一流 144 所也已打标签）
- 项目库：全澳洲 2920 个授课型硕士（CRICOS）；录取要求只有 9 个示例项目有，其余显示「未知（待采集）」
- 看原型：`cd ../offerpilot-prototype/web && npm run build && npx next start -p 3015`，打开 `/prototype`；
  或 Vercel 预览 web-git-codex-static-prototype-mortyyjts-projects.vercel.app（需登录 Vercel）

## 下一步

1. MortyYJT 继续看原型、提修改意见。待议：剩 59 所没有校徽；298 个常用译名需要人工核对
2. 原型定稿后：按已定的后端决定拆成 OpenSpec change 并排期（CRICOS 骨架、三档核验、官方页采集 + Claude API
   抽取、规则分档、留资与转人工），派给 Codex
3. PR #16 合并后，按 dispatch 第 7 步给 dsh 派出题

## 待用户回答（之前留下的）

- `.git/config` 的本地身份是占位的 `rehearsal`，要不要改回 `MortyYJT` + noreply（本会话每次提交都用 `-c` 参数覆盖）
- 是否在 GitHub 设置里隐藏个人邮箱（网页合并产生的 merge commit 用的是个人邮箱）
- 原型示例门槛偏高，要不要调低

## 坑

- Codex 沙箱：在 worktree 里提交不了（`--add-dir` 实测无效），不能联网，不能监听端口。由 Claude 检查后提交，浏览器验收也由 Claude 做。
  Codex 越界改过 `note/handoff.md`，交付后要检查改动范围
- 指南者（403）和 QS 的接口（403）都拒绝程序访问，**不要绕过**；QS 名次是用浏览器正常浏览取到的
- QS 名次已提交进公开仓库（MortyYJT 接受风险）；**商用前必须换源**（见 backlog）
- Wikidata 限流（429）：脚本要退避重试，调用间隔 ≥2 秒
- dsh 会话：带 `session-` 前缀的文件夹才是对话，纯 UUID 的是子 agent
- 构建数据和抓取脚本在会话临时目录里，没有进仓库。要重建数据，就按 PR #16 的提交说明重新抓取
- 其余旧坑见上一版 handoff（`794a911`）

## 怎么跑

- `make dev` → `make verify`；只看前端：`cd web && npm test && npm run build`
- 工作树：`../offerpilot-workflow-v2`（PR #13）、`../offerpilot-prototype`（PR #16）
