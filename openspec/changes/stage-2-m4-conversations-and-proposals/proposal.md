# stage-2-m4-conversations-and-proposals

## Why

M1–M3 把档案、选校组合、申请进度、路线图、材料库与审核依据都搬到了服务端，但**「留学顾问」这个标签页至今是空的**：`AdvisorView.tsx` 只有一个占位空态。上位设计 §4.6 为它规划的四张表（`conversations`、`messages`、`agent_runs`、`proposals`）一张都还没有。

同时有两件事在等这张表：

1. **agent 的写操作没有落点。** 上位设计 §3.5 定了"写操作分级确认"：低风险直接执行（M2b 的勾选、M3 的归档已经各自有路由了），高风险必须落成一条 `proposals` 行、等用户确认才生效。现在没有 `proposals`，所以任何"顾问建议你改申请组合"这样的动作都无处安放。
2. **M3 欠了一笔账。** M3 的 `document_reviews.run_id`（这条审核结论是哪次 agent 执行产生的）明确推迟到 M4——因为 `agent_runs` 表当时不存在（归档的 design.md D10）。

按 `note/product/roadmap.md`，**接模型是后面单独一项**（第 3 项）。所以 M4 的目标不是"让顾问会说话"，而是**把说话的通道、留痕与确认机制建好**，并让它在没有模型的情况下也能被端到端验证：申请人可以读、可以自己发消息；顾问一侧的消息与提案暂时由维护者命令写入，并**明确标注来源是人而不是模型**。模型接进来时（第 3 项），换掉的是"谁写这条消息"，不是表结构。

## What Changes

- 新增 `conversations` 与 `messages`：一条会话挂多条消息，`seq` 在会话内连续（`UNIQUE (conversation_id, seq)`，写入时对会话行加锁，与 M3 的版本号同一套做法）。
- 新增 `proposals`：待确认的改动（`kind` / `payload` / `risk` / `status`），申请人可以确认或拒绝，**确认时在一个事务里套用既有服务**（改档案、改路线、改组合、建任务、归档材料）。
- 新增 `agent_runs`，并补上 M3 推迟的那条外键 `document_reviews.run_id → agent_runs.id`（可空）。**在模型接进来之前，没有任何代码会写 `agent_runs`**：一行 run 意味着"某个模型真的跑过"，在没有模型的时候写它等于伪造来源。
- 申请人侧 HTTP：读会话列表、读一条会话的消息、发一条自己的消息、读待确认提案、确认 / 拒绝一条提案。
- 维护者命令 `api/advisor_cli.py`：写入一条**顾问侧**消息（`metadata.origin = "operator"`）与一条提案。这是模型接进来之前唯一的写入路径，和 M3 的 `review_cli.py` 是同一个道理——仓库没有登录与角色，能产生判断的写入不开 HTTP。
- 界面：`AdvisorView` 从占位空态变成真的对话视图——消息列表、输入框、待确认提案卡片（确认 / 拒绝）。顾问侧消息在界面上**标明来源**，不假装是模型说的。
- 提案过期：确认前检查"这份提案所依赖的东西在它生成之后有没有变"，变了就拒绝（409）并把提案标成 `expired`，而不是把一个基于旧档案的改动套上去。

**不做**：调用任何模型、prompt、工具编排、向量检索、流式输出（第 3 项）；数据标注（第 2 项）；账号登录；消息编辑与删除；多轮工具调用的可视化。

## Capabilities

### New Capabilities

- `conversation-thread`：会话与消息的存储、顺序、申请人自己的读写，以及"这条顾问消息是人写的还是模型写的"这件事必须可读。
- `write-proposals`：提案的创建、读取、确认、拒绝与过期，以及确认时如何安全地套用既有服务。
- `agent-run-record`：`agent_runs` 的结构与 `document_reviews.run_id` 这条外键，以及"没有模型就不写 run"的规则。

### Modified Capabilities

- `document-review`：`document_reviews` 增加可空的 `run_id`（改自 M3 归档 spec 的同一节：当时这一列被明确推迟）。这是唯一一处对既有 current-truth spec 的修改。

## Impact

- **数据库**：一次新迁移，新建 4 张表（`conversations`、`messages`、`agent_runs`、`proposals`）并给 `document_reviews` 加一列可空外键。不改动既有列，不删任何东西。
- **后端**：`api/app/models/{conversation,proposal}.py`、`api/app/services/{conversations,proposals}.py`、`api/app/routers/{conversations,proposals}.py`、`api/app/schemas/*`、新命令 `api/advisor_cli.py`；`main.py` 各挂一行。
- **前端**：`web/components/AdvisorView.tsx` 改为真实对话视图；新增适配层（对照真实响应体）与 `web/lib/api.ts` 的读写函数；`page.tsx` 接线。
- **复用而非重写**：确认提案时套用的服务是既有的 `app/services/{writes,roadmap_tasks,applications,documents}.py`——提案只描述"要改什么"，怎么改仍然是那些服务说了算。
- **测试**：`api/tests/test_*conversation*` / `test_*proposal*` / `test_advisor_cli.py`；前端适配层单测对照抓下来的响应体；走查新增"顾问页读到一条顾问消息 + 确认一条提案"的步骤与截图。
- **检查**：`scripts/check-no-fake-dates.cjs` 不受影响；走查的清理要带上新的主体级联（`conversations` 与 `proposals` 都挂在 `clients` 上，CASCADE）。
