# 09. 自修复闭环与数据飞轮

> 目标：理解失败用例如何自动诊断、生成修复建议、沙箱重跑，并沉淀为训练数据。

## 1. 什么是测试自修复？

测试自修复不是“失败了就绕过”，而是：

```text
在确认失败原因属于用例问题或环境波动时，自动生成可审计的修复建议，并通过沙箱重跑验证。
```

目标：

```text
减少维护成本
提升用例稳定性
沉淀失败经验
形成数据闭环
```

## 2. 可修复的问题

适合自动修复：

```text
等待时间不足
状态等待条件不精确
UI / Actor 定位变化
操作距离不足
前置条件缺失
录制转写参数错误
步骤粒度太粗
测试数据失效
```

不适合自动修复：

```text
核心业务逻辑失败
奖励未发放
崩溃
严重性能回归
权限错误
安全校验缺失
断言目标被删除
```

这些应该标记为产品缺陷或高风险问题。

## 3. Healer 工作流

```text
失败用例
  ↓
收集 Observation
  ↓
Diagnoser 判断失败类型
  ↓
Healer 生成 JSON Patch
  ↓
Sandbox 重跑
  ↓
通过：生成修复建议
失败：保留诊断记录
  ↓
人工确认
  ↓
合入用例库
```

## 4. JSON Patch 示例

原始失败：

```text
Step 6 interact RewardChest failed
原因：玩家距离宝箱过远
```

修复 Patch：

```json
{
  "patch_type": "insert_step",
  "case_id": "dungeon_reward_flow_001",
  "before_step_id": 6,
  "step": {
    "action": "move_to",
    "params": {
      "target": "RewardChest",
      "distance_lte": 120
    }
  },
  "reason": "原用例在交互宝箱前未确保玩家进入可交互距离"
}
```

## 5. 自修复 Guardrails

必须设置护栏：

```text
禁止删除核心断言
禁止随意 skip 用例
禁止扩大 timeout 到不合理范围
禁止修改业务预期
禁止隐藏产品缺陷
高风险 patch 必须人工审批
每次最多修复 N 轮
每轮必须保留 diff 和证据
```

## 6. 沙箱重跑

修复后的用例不能直接进入主库。

必须：

```text
临时生成修复版本
在隔离环境重跑
比较修复前后结果
验证没有删除关键断言
生成 patch report
等待人工确认
```

## 7. 数据飞轮是什么？

每次测试都会产生数据：

```text
需求
测试点
BDD
JSON
执行轨迹
失败日志
诊断结果
修复 Patch
人工反馈
最终用例
```

这些数据可以继续训练模型，使系统越来越强。

## 8. 数据资产结构

不要只保存：

```text
自然语言 → JSON
```

应该保存完整链路：

```json
{
  "requirement": "测试玩家完成副本并领取奖励",
  "test_points": [],
  "bdd": "Given ... When ... Then ...",
  "json_case": {},
  "execution_trace": [],
  "failure": {
    "type": "case_error",
    "root_cause": "缺少移动到宝箱前的步骤"
  },
  "healer_patch": {},
  "final_case": {},
  "human_review": {
    "accepted": true,
    "comment": "修复合理"
  }
}
```

## 9. 可训练的模型任务

```text
Planner SFT：需求 → 测试点
Generator SFT：BDD/Spec → JSON
Failure Classifier：Observation → 失败类型
Diagnoser：Trace/Log/State → 根因解释
Healer：失败上下文 → JSON Patch
Reward Model：判断用例质量
DPO：好修复 > 坏修复
```

## 10. 评测指标

```text
JSON 格式正确率
Schema 通过率
可执行率
人工采纳率
测试覆盖率
执行成功率
失败归因准确率
自修复成功率
误修复率
核心断言保留率
```

## 11. 一句话总结

> 自修复闭环的本质不是让 AI 掩盖失败，而是让 AI 在证据充分、风险可控的情况下修复测试资产。每一次生成、执行、失败、诊断、修复和人工反馈，都会进入数据飞轮，持续提升模型和测试平台能力。