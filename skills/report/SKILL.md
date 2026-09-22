---
name: report
description: 资深 QA 视角的专业测试报告生成。读取 .qa-powers/evidence/<run-id>/result.yaml（机器契约），汇总四态统计、资深 QA 业务画像覆盖度（压门槛/超额/封顶）、数据真实性与非空校验审计以及失败深度归因，输出标准化 Markdown 报告。用户说"生成报告"、"看看测试结果"时使用。
allowed-tools: Read, Write, Bash(ls:*), AskUserQuestion
---

# report：生成测试报告

## 1. 选 run

`ls .qa-powers/evidence/` 列出可选 run-id；用户没指定就用最新的一个。**版本核对**：`bash "$CLAUDE_PLUGIN_ROOT/scripts/version-check.sh" .qa-powers/config.yaml` 有输出则把警告转告用户（中文），流程继续（仅提示、不阻断）。

## 2. 读取（只读 result.yaml，不解析日志/截图）

- run 级 `evidence/<run-id>/result.yaml`：用例列表与 summary
- 每条 case 的 `evidence/<run-id>/<case-id>/result.yaml`：断言详情、失败证据引用

## 3. 生成 `.qa-powers/reports/<run-id>.md`

结构（固定）：

```markdown
# 测试报告 <run-id>

模块：<module>　　时间：<生成时间>

## 总览

| 状态 | 数量 |
|---|---|
| 总数 | 2 |
| PASS | 1 |
| FAIL | 1 |
| BLOCKED | 0 |
| SKIPPED | 0 |

## 用例明细

| 用例 | 状态 | 失败原因 |
|---|---|---|
| case-01 正常下单流程 | ✅ PASS | |

case result.yaml 带 `account:` 时，用例名后附账号（如 `case-02 下单 [buyer]`）；多账号 run 建议在总览后加一节「账号覆盖」：每个账号跑了哪些 case、通过率。

## 业务画像覆盖与数据真实性审计（资深 QA 黄金准则）

汇总本次测试所覆盖的业务数据画像，并核验业务逻辑穿透度：
- **画像 1（压门槛/0分临界画像）**：验证门槛扣减未漏扣，超额 0 分基线。
- **画像 2（首档有效超额画像）**：验证超额奖励/得分精确匹配计算公式。
- **画像 3（溢出打满封顶画像）**：验证系统防线截断，未无限膨胀，守护财务与业务底线。
- **存量历史兼容画像**：验证历史分期/老单据不受新指标污染。
- **【数据真实性与非空核验】**：所有 PASSED 用例均已穿透核实具体业务数值，未出现"暂无数据"或空值假阳性。

## 覆盖改动点与验证结论

从 `cases/<模块>/meta.yaml` 的 `changes` 段拉取（id + desc，旧用例无 desc 时用 ref 兜底），按各 case 的 `covers` 关联：每个改动点列出覆盖它的 case 及其状态（✅ PASS / ❌ FAIL），给一句验证结论（结合该 case 断言的 actual 概括）。meta 无 `changes` 段（旧用例）时跳过本表。

| 改动点 | 覆盖用例 | 验证结论 |
|---|---|---|
| D1 结算页订单表单组件 | case-01 ✅ / case-02 ✅ | 正常下单成功、库存扣减一致 |

## 失败详情（每条 FAIL 一节）

预期/实际取该 case result.yaml 中 **status=failed 的断言**（一个 case 可能有多条断言，failed 的逐条列出，不要拿 passed 的断言充数）；db 断言带 `carrier:` 时在断言行标注载体（usql / runner）。case result.yaml 的 cleanup 段有记录（如「执行中误创建并已清理」）时，在该节末尾如实注明。

### case-02 库存扣减验证 ❌

- **步骤**: step 8（提交订单）
- **断言 1**: 预期 UI 出现"下单成功"｜实际 出现"系统异常"
- **断言 2**: 预期 DB orders 新增 1 行｜实际 未新增（多条 failed 断言逐条列出）
- **证据**: `evidence/<run-id>/case-02/screenshots/step-08.png`（tracing 见 run 会话）
- **覆盖改动点**: case result.yaml 的 covers（如 `backend:OrderController.create`）→ 初步判断方向：<结合预期/实际差异给一句话假设，如"提交接口报错，建议查后端日志与 OrderController.create" >

## BLOCKED 说明（有才写）

<case-id>: <reason>
```

## 4. 收尾

输出报告文件路径 + 一段话摘要（总数、通过率、最需要关注的失败）。提示：需要深入分析失败原因可继续对话排查（MVP 无独立 debug skill）。
