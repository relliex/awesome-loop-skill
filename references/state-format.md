# 状态与证据格式（.loop/ 完整规范）

主模型按本文件初始化和维护 `.loop/`。所有跨阶段、跨轮次的判断只依据
这里的落盘内容，不依赖聊天记忆。

## state.json 示例

```json
{
  "phase": "D",
  "phase_status": "running",
  "round": 3,
  "consecutive_clean": 1,
  "candidate_id": "cand-003",
  "candidate_base": "git a1b2c3d 或目录快照说明",
  "attack_surfaces": ["功能与用户路径", "边界与异常", "集成与数据"],
  "required_clean_rounds": 2,
  "gates": {
    "G1": {"status": "passed", "at": "2026-09-17", "rounds": 2,
           "record": ".loop/requirements/user-decisions.md",
           "user_words": "要成稿，能直接发；引用必须逐条附来源"},
    "G2": {"status": "passed", "at": "2026-09-17",
           "prd": "v1.2", "acceptance": "v1.1",
           "user_words": "接受这个方案"},
    "G3": {"status": "passed", "at": "2026-09-17", "team": "team.md v1.0",
           "user_words": "同意，就按这个团队来"}
  },
  "frozen": {
    "prd": "v1.2", "acceptance": "v1.1", "design": "v1.0（若启用）",
    "team": "v1.0", "runbook": "v1.0"
  },
  "blocked_on": [],
  "budget": {
    "dispatch_total": 80,
    "dispatch_used": 41,
    "fix_attempts_per_issue": 2,
    "round_limit": 6
  },
  "gate_p0": {
    "checked_at": "2026-09-17",
    "real_subagent_tool": "PASS",
    "probe_dispatch": "PASS（T-P0-PROBE-01 真实返回）",
    "file_io_and_command": "PASS",
    "loop_persistence": "PASS",
    "user_selector": "PASS",
    "verdict": "OK",
    "evidence": ".loop/evidence/p0-capability-gate.md"
  },
  "history": [
    {"round": 1, "verdict": "FOUND", "issues": ["ISSUE-001", "ISSUE-002"]},
    {"round": 2, "verdict": "FOUND", "issues": ["ISSUE-003"]},
    {"round": 3, "verdict": "CLEAN", "issues": []}
  ]
}
```

判定词表：对抗验证结论只有两个取值——`CLEAN`（未发现问题，必须附尝试
清单）与 `FOUND`（发现问题，必须附最小复现/证据）；`history[].verdict`
使用同一词表。连续通过计数（`consecutive_clean`）只统计全攻击面 `CLEAN`
的轮次。

与报告头 `status` 的关系：`status`（PASS / FAIL / BLOCKED）描述实例的
执行状态，`CLEAN` / `FOUND` 描述对抗验证的结论，两者独立——`status:
PASS` + 结论 `FOUND` 是正常组合（实例正常执行并发现了问题）。

门的规则：`status` 为 `passed` 且带用户原话要点与时间，才算通过；未记录
或含糊答复一律视为 `pending`，不得进入下一阶段（SKILL.md §3）。

## 团队编排（team.md）

G3 批准后冻结，版本化。每个角色都有独立产物与判据，禁止为热闹造角色。

```markdown
# 团队编排 v1.0（G3 批准 2026-09-17）

- 方案：prd.md v1.2 / acceptance.md v1.1
- 规模：8 个角色实例（含 3 名对抗验证者）
- 验证轮数：连续 2 轮全攻击面通过（高风险项 3 轮）
- 预算分摊：总 80 次派发，其中对抗验证 ≥ 40%

| ID | 原型 | 岗位 | 目标函数 | 信息边界 | 对抗配对 | 产物 |
|----|------|------|----------|----------|----------|------|
| PM-1 | producer | 需求与判据 | 产出可判定的验收标准 | 任务输入、访谈记录 | QA-1 | acceptance.md |
| DEV-1 | producer | 前端实现 | 让前端部分通过验收 | prd、判据、写范围 | QA-2 | 代码 + 报告 |
| DEV-2 | producer | 后端实现 | 让服务端部分通过验收 | prd、判据、写范围 | QA-2 | 代码 + 报告 |
| QA-1 | adversary | 判据攻击者 | 证明验收标准不可判定/有歧义 | 判据原文 | — | 问题清单 |
| QA-2 | adversary | 功能对抗验证者 | 用实际运行证明不达标 | 判据 + runbook | — | 验证报告 |
| DOC-1 | producer | 文档 | 让文档与真实行为一致 | 最终工件、runbook | QA-3 | 文档 |
| QA-3 | adversary | 盲读者 | 证明按文档无法完成操作 | 文档 + runbook | — | 验证报告 |
| AUD-1 | auditor | 独立审计 | 验证追踪链完整 | 全部索引与证据 | — | 审计报告 |

## 派发顺序与依赖

1. PM-1 →（冻结）→ QA-1 攻击判据 →（修复判据）→ G2
2. DEV-1 ∥ DEV-2（写范围互不重叠）
3. QA-2（并行攻击面：功能 / 边界 / 集成）
4. 问题返工：DEV → QA-2 复核 → 候选重建 → 重开轮次
5. DOC-1 → QA-3 → AUD-1
```

## 挂起（PARKED）语义

遇到 `BLOCKED_CAPABILITY`、待用户决策或外部依赖时：

1. 落盘：`phase_status` 改为 `blocked`，门禁结果写 `gate_p0`，每个待
   决策项写入 `blocked_on`，写明 `next_action_when_unblocked`；
2. 报告并停止——不等待、不擅自降级、不角色扮演顶替；
3. 解阻后按"恢复检查单"恢复，不重派已有真实返回的任务。

`blocked_on` 条目 schema：

```json
{
  "id": "BLK-001",
  "kind": "BLOCKED_CAPABILITY | USER_APPROVAL_PENDING | USER_INPUT_PENDING | EXTERNAL_DEPENDENCY",
  "issue": "事实描述 + 已尝试什么 + 证据位置",
  "decision_needed_from_user": "需要用户决定什么（含可选项）",
  "consequence": "不解阻会导致什么"
}
```

## 候选与版本规则

- candidate_id 在工件发生任何改动后递增（cand-004…），旧证据失效；
- prd.md / acceptance.md / team.md（以及启用的 design/design.md）带版本
  号与日期页眉；冻结后只读；
- 变更流程：用户请求**先原话落盘 user-decisions.md** → 回方案/编排角色
  → 新版本 → G2/G3 重确认 → 候选作废 → 验证计数清零。

## 用户决策记录（requirements/user-decisions.md）

这是方案与团队编排的溯源依据，也是对抗"命令稀释"的原始档案：每次
选择器问答的用户答复都原话落盘，按门与轮次组织。

```markdown
# 用户决策记录

## G1 第 1 轮 澄清 2026-09-17（问题清单来自需求完整性分析）

- D-1 交付物形态
  - 问题：最终交付是"可直接发布的成稿"还是"供你继续加工的半成品"？
    选项 A 成稿（推荐）/ B 半成品 / C 两版都给
  - 用户原话："要成稿，能直接发"（选 A）
  - 生效：prd v1.2 FR-1；acceptance AC-01 同步
- D-2 引用密度
  - 问题：关键陈述是否必须逐条附来源？A 必须（推荐）/ B 仅数据类必须
  - 用户原话：（未答复，可默认项）→ 按 A 记录生效，冻结前可推翻

## G2 方案确认 2026-09-17
- 用户原话："接受这个方案，但预算压到 60 次"
- 生效：prd v1.2 → 预算 60；团队预算分摊随之调整

## G3 团队确认 2026-09-17
- 用户原话："同意，就按这个团队来"
- 生效：team.md v1.0 冻结
```

规则：

- 主模型只记录原话与生效范围，**不改写用户含义**——转述即稀释；
- 每个冻结文件版本页眉登记吸收的 D 编号（如 prd v1.2 ← D-1~D-6）；
- 循环中途用户的口头变更同样先落盘为新 D 条目，再走变更流程；
- 锚定协议与恢复检查单都包含本文件。

## 任务包（handoffs/{task_id}.md）完整字段

```yaml
task_id: T-D-VERIFY-R2-edge
role: QA-2
archetype: adversary          # producer | adversary | auditor
objective: 用实际运行证明边界与异常面不达标（服务 AC-05~AC-12）
anchors:
  - source: ".loop/requirements/acceptance.md v1.1"
    text: "AC-08（逐字摘录原文）：……"
  - source: ".loop/team.md v1.0"
    text: "QA-2 目标函数（逐字摘录原文）：……"
baseline: {candidate_id: cand-003, prd: v1.2, acceptance: v1.1, team: v1.0}
inputs:
  - .loop/requirements/acceptance.md
  - .loop/runbook.md
business_context: 本节所需最小背景，不含其他角色的结论
write_scope:
  - .loop/reports/verify/round2-edge.md
  - .loop/evidence/verify/round2-edge/
allowed_tools: [读文件, 执行命令, 运行交付物]
forbidden: [读 reports/ 下生产者报告, 修改被验工件, 修改判据]
required_outputs:
  - 报告路径: .loop/reports/verify/round2-edge.md
  - 必含: [尝试清单, 逐条判据实测, 问题最小复现, 结论]
budget: {time_minutes: 20, tool_calls: 60}
```

注意：生产者自评、既往轮次报告与问题清单**只允许出现在修复类任务包里**，
永远不出现在对抗验证者的任务包里（盲测）。对抗验证者的 anchors 只摘自
它被允许看到的冻结文件，盲测防火墙不受影响。

## 问题文件（issues/ISSUE-{id}.md）

```markdown
# ISSUE-001
- 发现: round 1 / QA-2 功能面 (T-D-VERIFY-R1-func)
- 严重度: 高 / 阻断: 是
- 关联: AC-05（持久化）
- 最小复现/证据: 执行 X 后立即 Y，期望 <a>，实际 <b>（证据: evidence/…）
- 责任角色: DEV-2
- 状态: fixed            # open / fixed / verified / closed
- 修复次数: 1
- 修复记录: T-D-FIX-01 报告
- 复核: T-D-REVERIFY-01 复现重放通过
- 关闭证据: round 3 全攻击面通过
```

## 各原型报告最小模式

所有报告共用头部：`task_id / role / archetype / input_baseline / status
(PASS | FAIL | BLOCKED) / 证据位置`。

- **生产者**：产物清单；自检方法与真实输出；未尽事项；
  unresolved_questions。
- **对抗验证者**：尝试过的验证与攻击清单；逐条判据实测结果；问题列表
  （最小复现/证据）；结论 `CLEAN`（附尝试证据）或 `FOUND`（附复现）。
- **审计者**：需求→工件→证据 追踪表（双向）；断裂点；结论。

## 证据绑定

每条证据必须可追溯到：工作项、实例 task_id、candidate_id、生成方法或
命令。证据形态随领域而变——命令输出、引用出处与页码、独立复算记录、
盲读者/盲用户回答原文、实测数值、截图。明确区分：已通过 / 已失败 /
未执行 / 环境阻塞 / 不适用。禁止伪造调用、日志、引用与实测结果；证据
中不得包含密钥或真实敏感数据。

## 恢复（中断后重启）检查单

恢复前依次核对并写入 state.json：

0. 锚定重读：`state.json` → `team.md` → `prd.md` → `acceptance.md`
   →（若启用）`design/design.md`；检查 `user-decisions.md` 是否有中断
   期间新增的用户决策——有则先走变更流程再继续循环；
1. 三道门的状态：G1/G2/G3 是否都已 passed 且记录了用户原话；
2. 工作区是否被外部修改过（对照 candidate_base）；
3. 冻结文件版本是否仍有效；
4. 运行中的派发是否已有真实返回（拿结果，不重派）；
5. 当前候选一致性（工件与 candidate_id 是否匹配）；
6. 已有证据是否属于当前候选；
7. 剩余预算。
