# 任务：阶段二 M4 会话、消息与提案

批次划分对应 `design.md` 的 D12：第 1–6 组是 PR 1（后端），第 7 组是 PR 2（前端），第 8 组是收尾。
标注 `TDD:` 的条目先写失败测试再写实现。**开工前必须先得到用户对 Open Questions 的答复**（尤其第 1 条：
申请人能否自己发消息）——那一条决定第 2 组与第 7 组各少不少一块。

## 1. 数据模型与迁移

- [ ] 1.1 新增 `api/app/models/conversation.py`：`conversations`（client_id CASCADE、title、created_at/updated_at）与 `messages`（conversation_id CASCADE、`UNIQUE (conversation_id, seq)`、role、content、metadata JSONB、created_at）；**不声明 ORM 关系**，查询一律显式 `select()`（与 M3 的两张 document 表同一理由）
- [ ] 1.2 role 的取值用 CHECK 约束钉住（user / assistant / system），metadata 可空
- [ ] 1.3 新增 `api/app/models/proposal.py`：`proposals`（client_id CASCADE、conversation_id CASCADE、message_id 可空、kind CHECK、payload JSONB、risk CHECK、status CHECK 默认 pending、resolved_by/resolved_at 可空、created_at）
- [ ] 1.4 新增 `api/app/models/agent_run.py`：`agent_runs`（message_id、workflow_version、prompt_version、provider、model、latency_ms、input_tokens、output_tokens、tool_calls JSONB、confidence JSONB、created_at），字段按上位设计 §4.6
- [ ] 1.5 手写迁移：先建 `agent_runs`，再给 `document_reviews` 加可空 `run_id` 外键（还 M3 的账），最后建另外三张表；`downgrade` 按依赖倒序干净回退
- [ ] 1.6 在 `api/app/models/__init__.py` 里导出新模型（漏了不会让测试变红，但下一次 `make revision` 会静默漏表）
- [ ] 1.7 TDD: `api/tests/test_conversation_model.py` — 无 client_id 的会话被拒；重复 `(conversation_id, seq)` 被拒；role / kind / risk / status 越界被拒；`document_reviews.run_id` 指向不存在的 run 被拒
- [ ] 1.8 TDD: 同文件 — 删除 `clients` 时会话、消息、提案级联消失（`conftest` 的主体清扫依赖它）
- [ ] 1.9 `make migrate` 从空库一次建全，`alembic downgrade -1` 再 `upgrade head` 干净往返

## 2. 会话与消息（后端）

- [ ] 2.1 新增 `api/app/services/conversations.py`：`load_or_create_conversation(session, client_id) -> Conversation`（首次写入时建，标题固定为「与留学顾问的对话」）
- [ ] 2.2 同文件：`append_message(session, conversation, *, role, content, origin) -> Message`，`seq` 在**会话行锁**下取（`SELECT ... FOR UPDATE`），并发追加不跳号；role=assistant 时 `origin` 必填
- [ ] 2.3 新增 `api/app/schemas/conversation.py` 与 `api/app/routers/conversations.py`：`GET /api/conversations`（本主体，最新在前）、`GET /api/conversations/{id}`（含按 seq 排序的消息）、`POST /api/conversations/{id}/messages`（申请人自己发）
- [ ] 2.4 TDD: `api/tests/test_conversations_api.py` — 空会话返回空消息；申请人发的消息 `role=user` 且按序返回；跨主体读会话 404；`DELETE` 405
- [ ] 2.5 TDD: 同文件 — assistant 消息缺 `origin` 被拒绝；`origin` 原样回读；跨主体读单条消息 404
- [ ] 2.6 TDD: 同文件 — 两个线程同时追加消息，落库的 seq 连续且无重复（与 M3 版本号竞态同一套测试写法）
- [ ] 2.7 `app/main.py` 挂上 conversations 路由

## 3. 提案的创建与读取（后端）

- [ ] 3.1 新增 `api/app/services/proposals.py`：`propose(session, conversation, *, kind, payload, risk, summary) -> Proposal`；校验 kind 取值、payload 不含服务端自有字段
- [ ] 3.2 同文件：`read_proposals(session, client_id) -> list[Proposal]` 与 `read_proposal(session, client_id, proposal_id)`
- [ ] 3.3 新增 `api/app/schemas/proposal.py` 与 `api/app/routers/proposals.py`：`GET /api/proposals`、`GET /api/proposals/{id}`（只读；创建不开放 HTTP）
- [ ] 3.4 TDD: `api/tests/test_proposals_api.py` — 列表与详情只服务本主体；跨主体 404；已解决的提案仍在列表里（拒绝不是删除）；`POST` / `DELETE /api/proposals` 405
- [ ] 3.5 TDD: 同文件 — kind 越界被拒；payload 里出现 `origin` / `status` / id / 时间戳被拒；`message_id` 指向别条会话的消息被拒
- [ ] 3.6 新增 `api/advisor_cli.py`（argparse，风格对齐 `review_cli.py`）：`say <conversation_id> --text …` 写入一条 assistant 消息（`origin="operator"`）、`propose <conversation_id> --kind … --payload … --risk … --summary …` 先写入承载它的那条 assistant 消息、再把 `proposals.message_id` 指向它（**关联只有这一个方向**，见 D9）、`list` 列出最近的会话与待确认提案
- [ ] 3.7 TDD: `api/tests/test_advisor_cli.py` — `say` 写入的 assistant 消息带 `origin=operator`；`list` 打印的会话 id 能被 `say` 接受（与 M3 `review_cli.py list` 同样的往返断言）

## 4. 确认、拒绝与过期（后端）

- [ ] 4.1 `POST /api/proposals/{id}/confirm`：D6 的过期检查 → 按 kind 调用既有服务 → 同一事务内标记 `confirmed` 并记录 `resolved_by` / `resolved_at`
- [ ] 4.2 `POST /api/proposals/{id}/reject`：只改状态，不套用任何东西
- [ ] 4.3 TDD: `api/tests/test_proposal_resolution.py` — `update_application` 的确认走 `replace_applications` 且它的拒绝规则照旧生效；`create_task` 的确认留下 `task_events` 记录
- [ ] 4.4 TDD: 同文件 — 重复确认 409；拒绝后目标状态不变；确认一条已被拒绝的提案 409
- [ ] 4.5 TDD: 同文件 — 档案在提案之后被改过 → 确认 409 且提案变 `expired`，档案不受影响；没有变过 → 正常确认（四种 kind 各一条）
- [ ] 4.6 TDD: 同文件 — payload 被既有服务拒绝时返回映射过的 4xx、目标状态不变、提案仍是 `pending`
- [ ] 4.7 TDD: 同文件 — 任何 risk 的提案在被确认之前都不会被套用（D7）

## 5. agent_runs 与 M3 欠的外键

- [ ] 5.1 TDD: `api/tests/test_agent_runs.py` — 走完一条完整流程（会话 → 消息 → 提案 → 确认 → 记录一条审核结论）之后 `agent_runs` 仍为空（**这条是本批次的红线测试**）
- [ ] 5.2 TDD: 同文件 — `document_reviews.run_id` 默认为空；显式写一个存在的 run 可以；写一个不存在的被数据库拒绝
- [ ] 5.3 人工核验（CLI 写一条审核结论后读回）：`GET /api/documents/{id}` 的结论里 `runId` 为 `null`，界面不显示任何 run 信息

## 6. PR 1 验收与提交

- [ ] 6.1 `make verify` 全绿，输出留进提交信息与 PR 描述
- [ ] 6.2 独立审查（fresh-context 子代理）：只报影响正确性或本 change 需求的问题
- [ ] 6.3 按审查结论修复并重跑 `make verify`
- [ ] 6.4 分支 `dsh/stage-2-m4-conversations-backend`，按 Conventional Commits 提交并开 PR 1（不自行合并，等用户）

## 7. 前端（PR 2）

- [ ] 7.1 `web/lib/types.ts` 增加会话、消息、提案的域类型；新增 `web/lib/conversation-source.ts` 适配层（wire → 本侧命名 + 文案），对照**抓下来的真实响应体**做单测
- [ ] 7.2 顾问消息的来源显示规则：`operator` 显示「人工录入」、`agent` 显示「顾问生成」、未知显示「来源未知」，**任何情况下都不得把未知来源说成模型生成**
- [ ] 7.3 提案卡片文案按 kind 与 risk 说人话（例如 update_application →「这会替换你整个申请组合」），并显示 payload 的关键内容而不是一段 JSON
- [ ] 7.4 `web/lib/api.ts`：读会话、发消息、读提案、确认、拒绝；失败时读出后端自己的中文错误（与 M3 的做法一致）
- [ ] 7.5 `web/components/AdvisorView.tsx`：从占位空态改为真实对话视图——消息列表（时间、角色、来源标记）、输入框、待确认提案区（确认 / 拒绝 / 过期后的说明）
- [ ] 7.6 `web/app/page.tsx` 接线：读会话与提案、写入后重读、失败要有可见提示
- [ ] 7.7 前端单测：适配层对照 fixture；未知 origin 不读成模型；提案文案按 kind 正确
- [ ] 7.8 `scripts/e2e-walkthrough.cjs` 新增步骤：用 `advisor_cli.py` 写入一条顾问消息与一条提案 → 顾问页读到它、看到「人工录入」标记 → 确认提案 → 断言服务端的目标状态真的变了、提案变成已确认；截图写入 `docs/screenshots/`
- [ ] 7.9 `make verify` 全绿（含浏览器走查截图）
- [ ] 7.10 独立审查 + 修复；分支 `dsh/stage-2-m4-conversations-frontend`，开 PR 2（不自行合并）

## 8. 收尾

- [ ] 8.1 重写 `note/handoff.md`
- [ ] 8.2 `note/product/roadmap.md` 把 M4 标为已完成并填上 change id；`note/product/backlog.md` 记下本批次暴露的新想法
- [ ] 8.3 两个 PR 都合入后，`openspec archive stage-2-m4-conversations-and-proposals`
