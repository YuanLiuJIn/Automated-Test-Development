# 03. UE5 游戏自动化测试基础

> 目标：理解 UE5 游戏测试和普通 Web/API 测试有什么不同，以及 Gauntlet、UAT、TestController、性能采集各自负责什么。

## 1. 游戏自动化测试为什么特殊？

游戏测试比普通软件测试复杂，因为它涉及：

```text
实时渲染
复杂输入
物理模拟
AI 行为
异步加载
网络同步
动画状态
大量 Actor
复杂 UI
性能波动
多平台差异
```

普通 Web 测试只需要判断 DOM 和接口返回，而游戏测试需要判断：

```text
玩家在哪
怪物是否生成
任务状态是否变化
UI 是否出现
帧率是否稳定
地图是否加载完成
角色是否卡死
Actor 是否销毁
```

## 2. UE 自动化测试常见层次

```text
C++ 单元测试
  验证底层逻辑：伤害公式、技能冷却、背包计算

Functional Test
  验证关卡和玩法片段：进入触发区、Actor 状态变化

Gauntlet / UAT 系统测试
  启动真实游戏 Build，跑完整流程，收集结果

性能测试
  固定场景、固定路线、固定压力条件下采集性能指标

稳定性测试
  长时间运行，检查崩溃、卡死、内存泄漏
```

## 3. Functional Test 是什么？

Functional Test 更接近 UE Editor 内部的功能测试。

它适合：

```text
在关卡里放测试 Actor
验证 Actor 是否触发事件
验证 UI 是否出现
验证某个玩法片段
本地快速调试
```

但它不一定适合完整 CI/CD，因为很多系统测试需要启动打包后的真实 Build。

## 4. Gauntlet 是什么？

Gauntlet 是 Unreal 的自动化运行框架。

它擅长：

```text
启动游戏 Build
启动客户端 / 服务端
传递命令行参数
控制测试生命周期
收集日志和结果
支持多平台执行
接入 CI/CD
```

可以把 Gauntlet 理解成：

```text
UE 游戏自动化测试的外部调度器。
```

它不一定替你写测试逻辑，但它负责把游戏拉起来并组织测试运行。

## 5. UAT 是什么？

UAT 全称是 Unreal Automation Tool。

它是 UE 的自动化工具入口，常用于：

```text
构建
打包
运行自动化
运行 Gauntlet
部署到设备
收集结果
```

DevOps 流水线里经常通过 UAT 触发 UE 构建和测试。

## 6. TestController 是什么？

TestController 是游戏内部测试控制器。

它通常运行在 UE 游戏进程中，负责：

```text
读取测试配置
控制测试开始/结束
驱动游戏内动作
等待游戏状态变化
采集结果
通知外部测试框架
```

在你的项目中，TestController 可以作为 UE C++ 端的核心模块：

```text
Python 发命令 → TestController 接收 → 游戏内执行 → 状态回传 Python
```

## 7. 为什么需要 UE C++ 状态回传？

游戏自动化不能只靠截图和日志。

更稳定的方式是 UE 主动返回结构化状态：

```json
{
  "level": "Dungeon_001",
  "phase": "Combat",
  "quest_state": "InProgress",
  "player_location": [120, 200, 0],
  "ui_visible": ["QuestPanel", "SkillBar"],
  "fps": 58.2
}
```

这样 Python 测试框架才能明确知道：

```text
地图是否加载完成
玩家是否进入目标区域
任务是否完成
UI 是否出现
性能是否异常
```

## 8. UE 性能采集基础

常见性能指标：

```text
FPS
Average Frame Time
P95 / P99 Frame Time
Game Thread Time
Render Thread Time
GPU Time
Memory Peak
Hitch Count
Load Time
Draw Calls
Actor Count
Crash Count
```

采集方式可以包括：

```text
UE CSV Profiler
Unreal Insights
日志解析
自定义 C++ 指标上报
外部性能采集工具
```

## 9. 游戏测试中常见断言

```text
状态断言：QuestState == Completed
位置断言：Player within target radius
Actor 断言：BossDead == true
UI 断言：SettlementPanel visible
背包断言：RewardItem count >= 1
日志断言：No Error / Ensure / Crash
性能断言：P95 FrameTime <= 33ms
稳定性断言：运行 1 小时无崩溃
```

## 10. UE 自动化测试的推荐架构

```text
DevOps
  ↓
Python Test Framework
  ↓
UAT / Gauntlet
  ↓
Game Build
  ↓
UE TestController
  ↓
Game State / UI / Actor / Perf Observer
  ↓
Result / Trace / Report
```

## 11. 一句话总结

> UE5 游戏自动化测试的关键，是把“外部调度”和“游戏内部状态”打通。Gauntlet/UAT 负责拉起和管理游戏进程，Python 负责测试编排和产物管理，UE C++ TestController 负责执行游戏内动作和返回结构化状态。