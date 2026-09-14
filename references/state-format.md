# 状态与证据格式（.loop/ 完整规范）

主模型按本文件初始化和维护 `.loop/`。所有跨阶段、跨轮次的判断
只依据这里的落盘内容，不依赖聊天记忆。

## state.json 示例

```json
{
  "phase": "P5",
  "phase_status": "running",
  "round": 3,
  "consecutive_clean": 1,
  "candidate_id": "cand-003",
  "candidate_base": "git a1b2c3d 或目录快照说明",
  "attack_surfaces": ["功能与用户路径", "边界与异常", "集成与数据"],
  "required_clean_rounds": 2,
  "blocked_on": [],
  "budget": {
    "dispatch_total": 80,
    "dispatch_used": 41,
    "fix_attempts_per_defect": 2,
    "round_limit": 6
  },
  "approvals": {
    "acceptance": "v2 已由用户批准 2026-09-11",
    "design": "v1 冻结"
  },
  "gate_p0": {
    "checked_at": "2026-09-11",
    "real_subagent_tool": "PASS",
    "probe_dispatch": "PASS（T-P0-PROBE-01 真实返回）",
    "file_io_and_command": "PASS",
    "loop_persistence": "PASS",
    "verdict": "OK",
    "evidence": ".loop/evidence/p0-capability-gate.md"
  },
  "history": [
    {"round": 1, "verdict": "FOUND", "defects": ["DEF-001", "DEF-002"]},
    {"round": 2, "verdict": "FOUND", "defects": ["DEF-003"]}
  ]
}
```

## 挂起（PARKED）语义

遇到 `BLOCKED_CAPABILITY`、待用户决策或外部依赖时：

1. 落盘：`phase_status` 改为 `blocked`，门禁结果写 `gate_p0`，
   每个待决策项写入 `blocked_on`，写明 `next_action_when_unblocked`；
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

- candidate_id 在代码发生任何改动后递增（cand-004…），旧证据失效；
- acceptance.md / design.md 带版本号与日期页眉；冻结后只读；
- 变更流程：用户请求**先原话落盘 user-decisions.md** → 回 PM →
  新版本 →（关键项）用户批准 → 候选作废 → 计数清零。

## 用户决策记录（requirements/user-decisions.md）

这是 PRD 的溯源依据，也是对抗"命令稀释"的原始档案：访谈中用户的
每一句答复都原话落盘，按轮次组织。

```markdown
# 用户决策记录

## 第 1 轮 访谈 2026-09-13（由 T-P1-PM-01 问题清单发起）

- D-1 数据文件位置
  - 问题：数据 JSON 存哪里？选项 A 当前目录 .todo.json / B 用户主目录
  - 用户原话："放当前目录吧"
  - 生效：PRD v1.1 FR-2；AC-03 同步修改
- D-2 重复 done 的幂等性
  - 问题：重复执行 todo done 3 应该报错还是静默成功？
  - 用户原话：（未答复，可默认项）
  - 生效：按默认建议"静默成功，退出码 0"记录生效；用户冻结前
    可随时推翻
```

规则：

- 主模型只记录原话与生效范围，**不改写用户含义**——转述即稀释；
- prd.md 每个版本页眉登记吸收的 D 编号（如 v1.1 ← D-1~D-6）；
- 用户在循环中途的口头变更同样先落盘为新的 D 条目，再走变更流程；
- 锚定协议与恢复检查单都包含本文件：需求判断以它 + 冻结 PRD 为准，
  不以对话记忆为准。

## 任务包（handoffs/{task_id}.md）完整字段

```yaml
task_id: T-P5-QA-R2-edge
role: QA
objective: 用实际运行证明项目在边界与异常面不达标（服务 AC-05~AC-12）
anchors:
  - source: ".loop/requirements/acceptance.md v1.0"
    text: "AC-08（逐字摘录原文）：todo done 9 对不存在的 id 应……"
baseline: {candidate_id: cand-003, acceptance: v2, design: v1}
inputs:
  - .loop/requirements/acceptance.md
  - .loop/runbook.md          # 如何构建、启动、调用项目
business_context: 极简待办 CLI，Python 3 标准库
write_scope:
  - .loop/reports/test/round2-edge.md
  - .loop/evidence/test/round2-edge/
allowed_tools: [读文件, 执行命令, 运行项目]
forbidden: [读 reports/ 下其他文件, 修改产品代码]
required_outputs:
  - 报告路径: .loop/reports/test/round2-edge.md
  - 必含: [攻击清单, 逐条验收实测, 缺陷最小复现, 结论]
budget: {time_minutes: 20, tool_calls: 60}
```

注意：`known_defects` 字段只允许出现在 FIX 的任务包里，
永远不出现在 QA 任务包里（盲测）。QA 的 anchors 只摘自
acceptance.md 与 runbook.md（其合法输入），盲测防火墙不受影响。

建议维护 `.loop/runbook.md`：项目的构建、启动、调用方法。它由
P3 的第一个 DEV 工作包产出，是 QA 实测的前提。

## 缺陷文件（defects/DEF-{id}.md）

```markdown
# DEF-001
- 发现: round 1 / QA-功能面 (T-P5-QA-R1-func)
- 严重度: 高 / 阻断: 是
- 关联: AC-05（持久化）
- 最小复现: `todo add` 后立即 `todo list`，期望显示新增条目，实际退出码 1
- 责任角色: DEV
- 状态: fixed   # open / fixed / verified / closed
- 修复次数: 1
- 修复记录: T-P6-FIX-01 报告
- 复核: T-P6-REV-01 复现重放通过
- 关闭证据: round 3 全攻击面 CLEAN
```

## 各角色报告最小模式

所有报告共用头部：`task_id / role / input_baseline / status
(PASS | FAIL | BLOCKED) / 证据位置`。

- **DEV**：改动文件清单；构建与自测命令及真实输出；未尽事项；
  unresolved_questions。
- **REV**：按严重度排序的问题清单（位置 + 依据）。
- **QA**：尝试过的攻击清单；逐条验收实测结果；缺陷列表（最小复现
  = 命令或步骤 + 期望 vs 实际）；结论 CLEAN（附尝试证据）或 FOUND。
- **FIX**：根因；改动；复现重放前后对比。
- **AUD**：验收项→实现→证据追踪表；断裂点；结论。

## 证据绑定

每条证据（文件、命令输出、日志）必须可追溯到：工作项、实例
task_id、candidate_id、生成命令或方法。明确区分：已通过 / 已失败 /
未执行 / 环境阻塞 / 不适用。禁止伪造调用、日志、覆盖率与部署结果；
证据中不得包含密钥或真实敏感数据。

## 恢复（中断后重启）检查单

恢复前依次核对并写入 state.json：

0. 锚定重读：`state.json` → `prd.md`（目标/范围/非目标）→
   `acceptance.md`；检查 `user-decisions.md` 是否有中断期间新增的
   用户决策——有则先走变更流程再继续循环；
1. 工作区是否被外部修改过（对照 candidate_base）；
2. 冻结文件版本是否仍有效；
3. 运行中的派发是否已有真实返回（拿结果，不重派）；
4. 当前候选一致性（代码与 candidate_id 是否匹配）；
5. 已有证据是否属于当前候选；
6. 剩余预算。

结果未知的有副作用操作（部署、迁移）先核查真实状态，禁止盲目重放。
