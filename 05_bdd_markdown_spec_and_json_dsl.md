# 05. BDD、Markdown Spec 与 JSON DSL

> 目标：理解为什么 AI 测试平台不要直接从自然语言跳到 JSON，而要增加可审查的测试规格中间层。

## 1. BDD 是什么？

BDD 全称：

```text
Behavior-Driven Development
行为驱动开发
```

它用接近自然语言的格式描述系统行为。

经典结构：

```gherkin
Given 前置条件
When  发生动作
Then  应该得到结果
```

例如：

```gherkin
Feature: 副本奖励流程

Scenario: 玩家完成副本后可以领取奖励
  Given 玩家已经接取副本任务
  And 玩家位于副本入口
  When 玩家进入副本
  And 玩家击败所有怪物
  And 玩家打开奖励宝箱
  Then 任务状态应该变为 Completed
  And 背包中应该新增副本奖励
  And 结算界面应该正常展示
```

BDD 的价值：

```text
人能读
测试能审
需求能追溯
模型能学习
后续能翻译成 JSON / 脚本
```

## 2. Markdown Spec 是什么？

Markdown Spec 是用 Markdown 写的测试规格文档。

它比 BDD 更自由，可以包含：

```text
测试目标
适用平台
前置条件
步骤
预期结果
风险点
性能指标
采集要求
上传要求
失败归因规则
```

示例：

```markdown
# 副本奖励流程测试

## 测试目标
验证玩家完成副本后，任务状态、奖励发放、结算 UI 和性能指标均符合预期。

## 前置条件
- 玩家已接取副本任务
- 背包至少有 1 个空位
- 平台：Win64
- 画质：High

## 测试步骤
1. 启动游戏并进入主城
2. 移动到副本入口
3. 进入副本
4. 击败所有怪物
5. 打开奖励宝箱
6. 返回结算界面

## 预期结果
- 任务状态为 Completed
- 背包新增 DungeonReward
- 结算 UI 可见
- 平均 FPS >= 45
- P95 FrameTime <= 33ms
- 无 Error / Crash 日志

## 数据采集
- FPS
- FrameTime
- Memory
- UE Log
- Screenshot
- Trace
```

## 3. 为什么需要 Spec 中间层？

如果直接：

```text
自然语言 → JSON
```

容易出现：

```text
测试点遗漏
步骤粒度不稳定
断言不明确
平台差异丢失
性能采集要求丢失
人类难以 Review
训练数据质量不可控
```

推荐链路：

```text
自然语言需求
  ↓
测试点列表
  ↓
BDD / Markdown Spec
  ↓
JSON DSL
  ↓
Python + UE 执行
```

## 4. JSON DSL 是什么？

DSL 全称：

```text
Domain-Specific Language
领域专用语言
```

JSON DSL 就是用 JSON 定义一套专门给 UE 自动化测试执行的“测试语言”。

它不是普通 JSON，而是有固定 schema 的执行协议。

例如：

```json
{
  "case_id": "dungeon_reward_flow_001",
  "title": "玩家完成副本并领取奖励",
  "priority": "P0",
  "platforms": ["Win64"],
  "steps": [
    {
      "id": 1,
      "action": "load_level",
      "params": { "level": "Dungeon_001" }
    },
    {
      "id": 2,
      "action": "move_to",
      "params": { "target": "DungeonEntrance" }
    },
    {
      "id": 3,
      "action": "assert_quest_state",
      "params": { "quest": "DungeonQuest", "expected": "Completed" }
    }
  ]
}
```

## 5. JSON DSL 的核心组成

推荐结构：

```text
case metadata：用例元信息
preconditions：前置条件
steps：执行步骤
assertions：断言
performance_budget：性能预算
collection：采集要求
cleanup：清理逻辑
traceability：追溯到哪个 Spec / 需求 / 缺陷
```

示例：

```json
{
  "case_id": "case_001",
  "source_spec": "specs/dungeon_reward.md",
  "preconditions": [],
  "steps": [],
  "performance_budget": {
    "avg_fps_gte": 45,
    "p95_frame_time_lte_ms": 33
  },
  "collection": {
    "logs": true,
    "screenshots": true,
    "trace": true,
    "upload_cloud": true
  }
}
```

## 6. Action 分类

可以把操作类型分成：

```text
环境类：load_level, reset_world, set_graphics_quality
角色类：spawn_player, move_to, attack, use_skill, interact
UI 类：click_ui, input_text, wait_ui, assert_ui_visible
状态类：wait_state, assert_actor_state, assert_quest_state
性能类：start_perf_capture, stop_perf_capture, assert_perf_budget
诊断类：screenshot, dump_actor_tree, dump_ui_tree, collect_logs
```

## 7. 状态等待 vs 时间等待

不推荐：

```json
{ "action": "sleep", "params": { "seconds": 10 } }
```

推荐：

```json
{
  "action": "wait_state",
  "params": {
    "state": "LevelLoaded",
    "timeout_sec": 30
  }
}
```

自动化测试稳定性的核心原则：

```text
少用固定 sleep，多用明确状态等待。
```

## 8. JSON Schema 校验

所有 JSON DSL 都应该用 schema 校验。

要校验：

```text
action 是否存在
参数类型是否正确
必填字段是否缺失
timeout 是否合理
平台是否支持
资源是否存在
断言是否明确
```

## 9. Spec 到 JSON 的生成流程

```text
Markdown Spec / BDD
  ↓
解析测试目标、前置条件、步骤、断言、采集要求
  ↓
匹配 action vocabulary
  ↓
生成 JSON DSL
  ↓
Schema 校验
  ↓
资源校验
  ↓
沙箱试跑
```

## 10. 一句话总结

> BDD / Markdown Spec 是人机共读的测试规格，JSON DSL 是机器可执行的测试协议。AI 测试平台应该先生成可审查的测试规格，再生成可校验、可执行、可追溯的 JSON 用例，而不是直接从自然语言一步跳到执行脚本。