# 00. 学习路线与项目全图

> 目标：把“AI + UE5 游戏自动化测试开发”拆成可学习、可实现、可展示的工程项目。

## 1. 项目最终定位

本项目最终形态：

```text
AI-native UE5 游戏智能自动化测试平台
```

它不是单纯“让大模型生成测试用例”，而是一个完整闭环：

```text
需求 / 录制 / 历史缺陷 / 代码变更 / 性能基线
  ↓
测试规格生成：测试点、BDD、Markdown Spec
  ↓
结构化用例生成：JSON DSL
  ↓
执行编排：Python 测试框架 + DevOps 流水线
  ↓
游戏内执行：UE C++ Test Plugin / TestController / 状态机
  ↓
多源观测：日志、截图、UI、Actor、Gameplay State、性能 Trace
  ↓
智能诊断：失败归因、性能回归、可疑 Commit 定位
  ↓
自修复：JSON Patch、沙箱重跑、人工确认
  ↓
数据闭环：SFT / DPO / 评测集 / 回归用例库
```

## 2. 需要掌握的基础知识

| 模块 | 需要学什么 | 对应文件 |
|---|---|---|
| 自动化测试基础 | 测试金字塔、测试用例、断言、回归、CI/CD | `02_foundations_automated_testing.md` |
| UE5 游戏测试 | UE 自动化体系、Gauntlet、UAT、Functional Test、性能采集 | `03_ue5_game_testing_foundations.md` |
| 三端协作 | Python 测试框架、UE C++ 状态回传、DevOps 流水线 | `04_three_layer_architecture_python_ue_devops.md` |
| 测试规格 | BDD、Markdown Spec、JSON DSL、Schema 校验 | `05_bdd_markdown_spec_and_json_dsl.md` |
| AI 生成 | Planner、Reviewer、Generator、SFT、RAG、评测 | `06_llm_generation_pipeline.md` |
| 执行与观测 | 状态机执行、Observer、日志、截图、Trace、状态快照 | `07_execution_observation_diagnosis.md` |
| 性能回归 | FPS、FrameTime、Z-Score、环比、阈值、Commit Blame | `08_performance_regression_and_commit_blame.md` |
| 自修复闭环 | Healer、JSON Patch、沙箱重跑、数据飞轮 | `09_self_healing_and_data_flywheel.md` |
| 实践路线 | 分阶段做出可运行项目 | `10_practice_milestones.md` |
| 总体蓝图 | 最终平台模块、数据流、简历表述 | `11_project_final_blueprint.md` |

## 3. 推荐学习顺序

```text
第 1 阶段：先懂测试工程
  02 自动化测试基础
  03 UE5 游戏测试基础

第 2 阶段：理解三端架构
  04 Python / UE C++ / DevOps 三端协作

第 3 阶段：建立测试规格体系
  05 BDD / Markdown Spec / JSON DSL

第 4 阶段：引入 LLM
  06 Planner / Reviewer / Generator

第 5 阶段：实现执行闭环
  07 执行、观测、诊断
  08 性能回归与 Commit Blame
  09 自修复与数据飞轮

第 6 阶段：做成项目
  10 实践里程碑
  11 最终项目蓝图
```

## 4. 项目能力分层

```text
┌────────────────────────────────────────┐
│ 产品入口层                               │
│ Web / CLI / IDE Skill / OpenAPI          │
├────────────────────────────────────────┤
│ AI 智能层                                │
│ Planner / Reviewer / Generator / Healer  │
├────────────────────────────────────────┤
│ 测试规格层                               │
│ BDD / Markdown Spec / JSON DSL / Schema  │
├────────────────────────────────────────┤
│ Python 编排层                            │
│ 拉起游戏 / 组装用例 / 采集数据 / 上传报告 │
├────────────────────────────────────────┤
│ UE C++ 执行层                            │
│ TestController / 状态机 / 状态回传        │
├────────────────────────────────────────┤
│ DevOps 流水线层                          │
│ 构建 / 打包 / 分阶段测试 / 质量门禁        │
├────────────────────────────────────────┤
│ 数据资产层                               │
│ 用例库 / Trace / 性能基线 / 训练数据集     │
└────────────────────────────────────────┘
```

## 5. 最终要能讲清楚的问题

学完和做完后，你应该能回答：

```text
1. 为什么 UE 游戏自动化测试需要 Python / UE C++ / DevOps 三端协作？
2. BDD / Markdown Spec 为什么是自然语言和 JSON 用例之间的中间层？
3. LLM 在测试平台中应该负责什么，不应该负责什么？
4. JSON DSL 如何设计才能稳定执行、可校验、可演进？
5. UE C++ 端如何把游戏状态结构化返回给 Python？
6. Python 如何拉起游戏、下发步骤、采集产物、上传云端？
7. DevOps 流水线如何拆成冒烟、回归、性能、稳定性多阶段？
8. 性能异常如何从“报警”升级到“定位可疑改动”？
9. 失败用例如何自动诊断和自修复？
10. 每次生成、执行、失败、修复如何沉淀为训练数据？
```
