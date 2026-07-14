# 06. LLM 测试生成 Pipeline：Planner / Reviewer / Generator

> 目标：理解大模型在测试平台中如何分工，而不是用一个 Prompt 一步生成所有东西。

## 1. 为什么不能一步到位？

直接：

```text
需求 → LLM → JSON 用例
```

常见问题：

```text
模型臆测游戏机制
测试点遗漏
异常路径不足
断言不明确
格式不稳定
平台差异没处理
人类 Review 成本高
```

更稳定的方式是 Pipeline：

```text
需求 / 录制 / 缺陷 / 代码 Diff
  ↓
Planner：测什么
  ↓
Reviewer：漏了什么
  ↓
Generator：怎么测
  ↓
Validator：能不能执行
  ↓
Executor：跑起来
```

## 2. Planner：测试规划

Planner 负责从输入中生成测试点、风险点、BDD 草稿。

输入：

```text
自然语言需求
PRD
玩法说明
历史缺陷
代码 diff
已有用例
录制轨迹
```

输出：

```text
测试目标
主路径
异常路径
边界条件
性能风险
影响面
BDD 草稿
优先级
```

示例输出：

```json
{
  "test_points": [
    "玩家能正常进入副本",
    "怪物能正常生成",
    "击败怪物后任务状态更新",
    "宝箱奖励进入背包",
    "结算 UI 展示正确",
    "过程中平均 FPS 不低于阈值"
  ],
  "risk_points": [
    "背包满时奖励发放失败",
    "副本加载过程中断线",
    "Boss 死亡事件未触发任务完成"
  ]
}
```

## 3. Reviewer：覆盖率审查

Reviewer 不负责生成最终用例，而是检查 Planner 漏了什么。

审查维度：

```text
主流程覆盖
异常流程覆盖
边界条件覆盖
历史缺陷覆盖
平台差异覆盖
性能风险覆盖
UI 状态覆盖
数据状态覆盖
```

示例：

```json
{
  "missing": [
    "背包已满时领取奖励",
    "玩家死亡后复活继续副本",
    "网络断开后重连"
  ],
  "coverage_score": 0.72
}
```

## 4. Generator：生成 JSON DSL

Generator 负责把 BDD / Spec 转为结构化 JSON。

要求：

```text
严格遵循 JSON Schema
只使用已注册 action
断言必须明确
timeout 必须合理
每一步可追溯到 Spec
```

输入：

```text
Markdown Spec / BDD
Action Vocabulary
JSON Schema
资源索引：地图、Actor、UI、道具、任务
```

输出：

```text
可执行 JSON Case
```

## 5. Validator：生成后的质量卡口

Validator 负责检查生成用例是否可执行。

校验：

```text
Schema 校验
Action 参数校验
资源存在性校验
平台支持校验
前置条件校验
断言有效性校验
风险规则校验
```

示例错误：

```text
action=interact 缺少 target
target=RewardChest 在地图 Dungeon_001 中不存在
assert_inventory 缺少 item 字段
timeout=9999 过大
```

## 6. 训练数据应该怎么构建？

你已有 SFT 数据集，建议从单一映射升级为多阶段数据。

不要只保存：

```text
自然语言 → JSON
```

应该保存：

```text
需求 → 测试点
测试点 → BDD
BDD → Markdown Spec
Spec → JSON
JSON → 执行 Trace
失败 Trace → 诊断
诊断 → 修复 Patch
```

这样可以训练多个子任务：

```text
Planner 模型
Generator 模型
Failure Diagnosis 模型
Healer 模型
Evaluator 模型
```

## 7. SFT、DPO、评测集分别是什么？

### SFT

Supervised Fine-Tuning，监督微调。

用于学习：

```text
输入需求 → 输出标准结果
```

### DPO

Direct Preference Optimization，直接偏好优化。

用于学习：

```text
好用例 > 坏用例
好诊断 > 差诊断
好修复 > 危险修复
```

### 评测集

用于固定评估模型能力，不能参与训练。

指标：

```text
JSON 格式正确率
Schema 通过率
可执行率
断言完整率
场景覆盖率
人工采纳率
执行成功率
失败诊断准确率
```

## 8. RAG 在生成中的作用

LLM 生成测试时，需要检索上下文：

```text
已有测试用例
游戏玩法文档
历史缺陷
地图/Actor/UI 资源表
Action Schema
性能指标说明
```

否则模型容易幻觉。

推荐：

```text
生成前先检索相关玩法、资源、历史用例
再让模型基于检索结果生成
```

## 9. 模型分级策略

不是所有任务都用大模型。

```text
大模型：需求理解、复杂规划、失败归因
小模型：格式转换、字段补全、简单分类
规则：Schema 校验、资源校验、风险拦截
```

这样可以降低成本，提高稳定性。

## 10. 一句话总结

> LLM 在测试平台中最好采用分阶段流水线：Planner 负责“测什么”，Reviewer 负责“漏了什么”，Generator 负责“怎么测”，Validator 负责“能不能跑”。这比一个 Prompt 直接生成 JSON 更可控、更可审计，也更适合构建高质量训练数据闭环。