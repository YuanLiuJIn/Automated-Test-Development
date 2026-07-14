# 11. AI-native UE5 游戏自动化测试平台最终蓝图

> 目标：把整个项目整理成可以学习、实践、答辩、写进简历的完整方案。

## 1. 项目一句话

```text
设计并实现一套 AI-native UE5 游戏自动化测试平台，支持需求/录制/历史缺陷到测试规格和 JSON 用例的自动生成，通过 Python + UE C++ + DevOps 三端协作完成真实游戏环境下的自动执行、状态观测、性能回归检测、失败归因、可疑提交定位和用例自修复，并将全过程沉淀为持续进化的测试数据资产。
```

## 2. 项目核心模块

```text
1. Test Spec Generator
   自然语言 / 录制 / 缺陷 → 测试点 / BDD / Markdown Spec

2. JSON Case Generator
   Spec → 结构化 JSON DSL

3. Python Test Orchestrator
   拉起游戏、组装用例、连接 UE、采集产物、上传报告

4. UE Test Plugin
   接收命令、执行动作、状态回传、性能采集

5. DevOps Pipeline Adapter
   构建、打包、分阶段测试、质量门禁

6. Observer System
   日志、截图、Actor、UI、Gameplay State、Trace、性能指标

7. Diagnosis System
   失败归因、性能异常解释、Commit Blame

8. Self-Healing System
   JSON Patch、沙箱重跑、人工确认

9. Dataset Curator
   SFT / DPO / 评测集 / 回归用例库
```

## 3. 端到端数据流

```text
输入：自然语言需求
  ↓
Planner 生成测试点
  ↓
Reviewer 补齐边界和风险
  ↓
生成 BDD / Markdown Spec
  ↓
Generator 生成 JSON DSL
  ↓
Validator 做 Schema 和资源校验
  ↓
DevOps 触发测试节点
  ↓
Python 拉起游戏并连接 UE
  ↓
UE TestController 执行步骤并回传状态
  ↓
Python 采集日志、截图、Trace、性能指标
  ↓
Diagnoser 做失败归因和性能异常解释
  ↓
Healer 生成修复 Patch 并沙箱重跑
  ↓
Curator 沉淀训练数据和回归用例
```

## 4. 关键技术点

### 4.1 测试规格中间层

```text
自然语言 → BDD / Markdown Spec → JSON DSL
```

价值：

```text
可审查
可追溯
可训练
可调试
可沉淀
```

### 4.2 三端协作

```text
DevOps：负责流水线调度
Python：负责外部编排和产物管理
UE C++：负责游戏内执行和状态回传
```

### 4.3 状态驱动执行

不依赖固定 sleep，而是：

```text
wait_until LevelLoaded
wait_until QuestState == Completed
wait_until UI visible
wait_until Actor exists
```

### 4.4 多源观测

```text
日志 + 截图 + UI树 + Actor状态 + Gameplay State + 性能曲线 + Trace
```

### 4.5 智能诊断

```text
失败分类
证据提取
根因解释
可疑 Commit 排序
修复建议
```

### 4.6 自修复闭环

```text
失败 → 诊断 → JSON Patch → 沙箱重跑 → 人工确认 → 合入用例库
```

### 4.7 数据飞轮

```text
每次生成、执行、失败、修复都会变成模型训练数据。
```

## 5. 推荐项目目录结构

```text
Automated-Test-Development/
├─ docs/                         # 学习与设计文档
├─ specs/                        # Markdown Spec / BDD
├─ schemas/                      # JSON Schema
├─ cases/                        # JSON 测试用例
├─ python_test_framework/         # Python 编排框架
│  ├─ cli.py
│  ├─ case_loader.py
│  ├─ game_launcher.py
│  ├─ ue_client.py
│  ├─ executor.py
│  ├─ artifact_manager.py
│  ├─ metrics_collector.py
│  ├─ reporter.py
│  └─ uploader.py
├─ ue_plugin/                     # UE C++ 测试插件设计/代码
│  ├─ TestCommandServer
│  ├─ TestController
│  ├─ ActionExecutor
│  ├─ StateObserver
│  └─ PerformanceCollector
├─ pipelines/                     # DevOps 流水线配置
├─ reports/                       # 测试报告样例
├─ artifacts/                     # 日志/截图/Trace 样例
├─ patches/                       # Healer 修复 patch
├─ datasets/                      # SFT/DPO/评测数据
└─ README.md
```

## 6. 简历表述升级版

```text
LLM 驱动的 UE5 游戏智能自动化测试平台：设计并实现一套 AI-native 游戏测试框架，构建“测试规格生成—JSON 用例编排—UE/Gauntlet 状态机执行—多源观测—失败诊断—性能回归报警—用例自修复”的闭环体系。平台支持自然语言、操作录制、历史缺陷和代码变更多来源生成测试规格，基于 SFT 模型将玩法目标转化为结构化测试 DSL，并通过 Python 测试框架与 UE C++ TestController 协同执行。系统采集 Actor 状态、UI 状态、Gameplay State、日志、截图、Trace 与性能指标，结合 Z-Score、环比、硬阈值和 Commit Blame 实现性能回归分钟级报警与可疑改动定位；进一步引入 Planner/Reviewer/Generator/Diagnoser/Healer 多 Agent 架构，实现测试设计、执行诊断、JSON Patch 自修复和训练数据回流。
```

## 7. 面试讲解顺序

建议按这个顺序讲：

```text
1. 游戏自动化测试痛点：复杂状态、跨平台、性能波动、维护成本高
2. 为什么引入 AI：测试设计、录制转写、失败诊断、自修复
3. 为什么不能让 LLM 直接操作游戏：需要确定性执行层
4. 三端架构：DevOps / Python / UE C++
5. 测试规格：BDD / Markdown Spec / JSON DSL
6. 执行链路：Python 拉起游戏，UE 返回状态
7. 观测体系：日志、截图、Actor、UI、性能 Trace
8. 性能回归：Z-Score + 环比 + 阈值 + Commit Blame
9. 自修复：失败归因 → JSON Patch → 沙箱重跑
10. 数据闭环：SFT / DPO / 评测集持续进化
```

## 8. 项目最大亮点

```text
不是“用 LLM 写测试脚本”，而是“用 AI 重构游戏测试平台的生产链路”。
```

具体亮点：

```text
1. 规格驱动，而不是脚本驱动
2. 三端协作，而不是单点脚本
3. 状态驱动执行，而不是 sleep 等待
4. 多源观测，而不是只看日志
5. 智能诊断，而不是只报失败
6. 自修复闭环，而不是人工维护
7. 数据飞轮，而不是一次性生成
```

## 9. 一句话总结

> 最终项目是一个面向 UE5 游戏的 AI-native 自动化测试平台：LLM 负责规划、生成、诊断和修复，Python 负责外部编排，UE C++ 负责游戏内确定性执行和状态回传，DevOps 负责流水线调度和质量门禁。它把游戏测试从“人工写脚本 + 人工看报告”升级为“规格驱动 + 自动执行 + 智能诊断 + 自修复 + 数据闭环”。