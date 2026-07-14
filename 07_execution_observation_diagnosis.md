# 07. 执行、观测与失败诊断

> 目标：理解测试用例如何真正执行、如何采集证据、如何判断失败原因。

## 1. 执行层的核心任务

执行层负责把 JSON DSL 变成真实游戏行为。

```text
JSON Step
  ↓
Python 下发命令
  ↓
UE C++ TestController 执行
  ↓
UE 返回状态
  ↓
Python 判断下一步
```

执行层追求：

```text
稳定
可复现
可观测
可诊断
可接入流水线
```

## 2. 状态机执行模型

建议每个用例有统一生命周期：

```text
INIT
  ↓
PREPARE_ENV
  ↓
LOAD_LEVEL
  ↓
RUN_STEP
  ↓
WAIT_CONDITION
  ↓
ASSERT
  ↓
COLLECT_OBSERVATION
  ↓
PASS / FAIL / TIMEOUT / CRASH
```

每个 step 都要记录：

```text
step_id
action
params
start_time
end_time
status
error
screenshot
logs
state_snapshot
```

## 3. 执行 step 的标准流程

```text
1. Python 读取 step
2. 校验 step 参数
3. 下发给 UE
4. UE 执行动作
5. UE 持续回传状态
6. Python 等待 expected condition
7. 成功则进入下一步
8. 失败则采集证据并进入诊断
```

示例：

```json
{
  "id": 6,
  "action": "interact",
  "params": { "target": "RewardChest" },
  "expect": { "state": "RewardPanelVisible" },
  "timeout_sec": 10
}
```

## 4. Observer：观测层

传统测试只知道 pass/fail。

AI-native 测试平台必须知道：

```text
失败时游戏处于什么状态
哪个 Actor 异常
哪个 UI 没出现
哪条日志报错
性能从哪里开始恶化
```

## 5. 需要采集哪些信息？

### 5.1 游戏状态

```text
当前地图
玩家坐标
玩家血量
玩家状态
任务状态
背包状态
Gameplay Tags
战斗状态
AI 状态
```

### 5.2 UI 状态

```text
Widget Tree
可见控件
可点击按钮
当前弹窗
加载中状态
焦点控件
可见文本
```

### 5.3 Actor 状态

```text
Actor 是否存在
Actor 类型
Actor 位置
Actor 是否可交互
Actor 生命周期
组件状态
```

### 5.4 性能状态

```text
FPS
FrameTime
GameThread
RenderThread
GPUTime
Memory
Hitch
LoadTime
```

### 5.5 运行证据

```text
UE Log
Screenshot
Video
Trace
Crash Dump
Network Log
Input Trace
```

## 6. Observation 数据结构

```json
{
  "case_id": "dungeon_reward_flow_001",
  "step_id": 6,
  "action": "interact",
  "status": "failed",
  "error": "Target RewardChest not interactable",
  "screenshot": "artifacts/case001/step006.png",
  "actor_state": {
    "RewardChest": {
      "exists": true,
      "interactable": false,
      "distance": 320
    }
  },
  "player_state": {
    "location": [100, 200, 0],
    "health": 100
  },
  "logs": [
    "RewardChest interaction rejected: player too far"
  ]
}
```

## 7. 失败归因分类

建议至少分为：

```text
product_bug：产品缺陷
case_error：用例错误
env_error：环境问题
locator_error：定位问题
wait_error：等待问题
data_error：测试数据问题
perf_regression：性能回归
crash：崩溃
flaky：非确定性波动
model_error：模型生成错误
```

## 8. 诊断输入

Diagnoser 输入：

```text
失败 step
JSON case
BDD / Spec
UE 状态快照
截图
日志
性能曲线
历史成功记录
最近代码变更
```

## 9. 诊断输出

```json
{
  "failure_type": "case_error",
  "root_cause": "用例在与宝箱交互前没有移动到可交互距离内",
  "evidence": [
    "RewardChest exists=true",
    "interactable=false",
    "player distance=320",
    "required distance < 150"
  ],
  "suggested_fix": {
    "type": "insert_step",
    "before_step_id": 6,
    "step": {
      "action": "move_to",
      "params": { "target": "RewardChest", "distance_lte": 120 }
    }
  },
  "confidence": 0.91
}
```

## 10. 诊断规则 + LLM 结合

不要完全依赖 LLM。

推荐：

```text
规则先判断明确问题：崩溃、超时、资源不存在、断言失败
LLM 负责综合解释：日志 + 状态 + 截图 + Spec
最后用风险规则限制修复范围
```

## 11. 一句话总结

> 执行层不只是把步骤跑完，更重要的是把每一步的游戏状态、UI 状态、Actor 状态、日志、截图和性能数据采集下来。只有观测足够完整，AI 才能做可靠的失败归因和用例自修复。