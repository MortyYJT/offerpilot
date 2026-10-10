# Session handoff

**This is the only current-state file. Overwrite it at the end of a session; read it at the start.**
Verify it against `git status`, `git log` and `gh pr list` before trusting it.

## Where we are（2026-10-11）

- `main` 在 `11d992e`：PR #13（流程 v2）、#14（M1–M3 复盘题）、#19（Playwright 改为从 `web/` 解析）、#20（CI 说明）都已合并
- 等 MortyYJT 合并的 PR：
  - PR #16 `codex/static-prototype`：静态原型 `/prototype`，最新提交 `ea2c91a`，CI 通过
  - `claude/docs-cleanup`：README 现状、`note/README.md` 目录、本文件、2026-10-11 的决定
- 产品定位：MortyYJT 留学中介的获客和初筛工具（见 `note/decisions.md`）
- 阶段：先用静态原型定功能点，再补后端
- 文档地图：先读 `note/README.md`

## 本地文件夹

- `留学agent/` 只是外层文件夹，仓库是 `留学agent/offerpilot`
- 同级的 `offerpilot-*` 是同一个仓库的 worktree：
  - `-prototype` 对应 PR #16
  - `-claude-paths` 对应已合并的 #19 和 #20，可以移除
  - `-workflow-v2` 对应已合并的 #13，可以移除
- `留学agent/` 顶层的 7 月旧项目（`web/ app/ db/ …`）已确认没用，提交都在 GitHub 上。删除命令已交给 MortyYJT 运行，是否运行了还没核实

## 原型现状（PR #16）

- 学校库 `/prototype/schools`：海外 346 所，按 QS 排序，有中文名和联盟标签；校徽 287 所
- 背景测评：本科院校从教育部 3167 所里搜索，自动识别 985 / 211 / 双一流
- 项目库：全澳 2920 个授课型硕士（CRICOS）；只有 9 个示例项目有录取要求，其余显示「未知（待采集）」
- 看原型：`cd ../offerpilot-prototype/web && npm run build && npx next start -H 127.0.0.1 -p 3015`，打开 `/prototype/schools`

## 下一步

1. MortyYJT 看原型、提修改意见（还没提）
2. 待议：59 所学校没有校徽；298 个常用译名要人工核对
3. 原型定稿后，按已定的后端决定拆 OpenSpec change 并排期，派给 Codex
4. PR #16 合并后，按 dispatch 第 7 步派 dsh 出题

## 待用户回答（等看完原型再问）

- `.git/config` 的本地身份还是占位的 `rehearsal`，要不要改回 `MortyYJT` 加 noreply 邮箱（目前每次提交都用 `-c` 覆盖）
- 要不要在 GitHub 设置里隐藏个人邮箱（网页上合并时产生的 merge commit 用的是个人邮箱）
- 原型里示例录取门槛偏高，要不要调低

## 坑

- Codex 沙箱：在 worktree 里提交不了，不能联网，不能监听端口。由 Claude 检查后提交，浏览器验收也由 Claude 做；交付后检查改动范围（它越界改过 `note/handoff.md`）
- 自动模式会拦截批量 `mv`、`rm` 本地文件，这类命令交给 MortyYJT 运行
- 指南者和 QS 的接口都返回 403，拒绝程序访问，**不要绕过**；QS 名次商用前必须换数据源
- Wikidata 限流（429）：调用间隔 ≥2 秒，失败要退避重试
- 构建原型数据的抓取脚本没有进仓库，要重建就按 PR #16 的提交说明重新抓
- 原型代码（`web/components/prototype/`）被压成了很长的单行，可读性差，以后要不要重排待定

## 怎么跑

- `make dev`，然后 `make verify`；只看前端：`cd web && npm test && npm run build`
