# Automated-Test-Development

> 系统学习并实践“AI + 自动化测试开发”，最终目标是做出一个可展示、可迭代、可写进简历的 **AI-native UE5 游戏智能自动化测试平台**。

## 1. 项目定位

本项目不是单纯“让 LLM 生成测试用例”，而是围绕 UE5 游戏测试构建完整闭环：

```text
需求 / 操作录制 / 历史缺陷 / 代码变更 / 性能基线
  ↓
BDD / Markdown Spec 测试规格
  ↓
结构化 JSON DSL 测试用例
  ↓
Python 测试框架编排
  ↓
UE C++ TestController 执行与状态回传
  ↓
DevOps 流水线自动跑测
  ↓
日志 / 截图 / Actor / UI / 性能 Trace 采集
  ↓
失败归因 / 性能回归 / Commit Blame
  ↓
用例自修复 / 数据集回流 / 模型迭代
```

## 2. 推荐阅读顺序

| 顺序 | 文件 | 内容 |
|---|---|---|
| 1 | `00_learning_path_and_project_map.md` | 学习路线与项目全图 |
| 2 | `02_foundations_automated_testing.md` | 自动化测试开发基础 |
| 3 | `03_ue5_game_testing_foundations.md` | UE5 游戏自动化测试基础 |
| 4 | `04_three_layer_architecture_python_ue_devops.md` | Python / UE C++ / DevOps 三端协作架构 |
| 5 | `05_bdd_markdown_spec_and_json_dsl.md` | BDD、Markdown Spec 与 JSON DSL |
| 6 | `06_llm_generation_pipeline.md` | LLM 测试生成 Pipeline |
| 7 | `07_execution_observation_diagnosis.md` | 执行、观测与失败诊断 |
| 8 | `08_performance_regression_and_commit_blame.md` | 性能回归检测与 Commit Blame |
| 9 | `09_self_healing_and_data_flywheel.md` | 自修复闭环与数据飞轮 |
| 10 | `10_practice_milestones.md` | 从 0 到 1 的实践里程碑 |
| 11 | `11_project_final_blueprint.md` | 最终项目蓝图与简历表述 |
| 12 | `01_ai_automation_testing_projects_research.md` | 公开项目与实践调研 |
| 13 | `references.md` | 参考资料与关键词索引 |

## 3. 需要补充的基础知识

```text
自动化测试基础：测试金字塔、用例、断言、回归、CI/CD
UE5 测试基础：Functional Test、Gauntlet、UAT、TestController、性能采集
测试规格：BDD、Gherkin、Markdown Spec、JSON DSL、Schema 校验
Python 编排：进程管理、命令行参数、WebSocket/HTTP、产物管理
UE C++ 插件：命令接收、状态观察、Actor/UI/GameState 回传
DevOps：流水线分阶段执行、参数传递、质量门禁、报告发布
AI 测试：Planner、Reviewer、Generator、Diagnoser、Healer、数据闭环
性能检测：FPS、FrameTime、Z-Score、环比、硬阈值、Commit Blame
```

## 4. 项目最终架构

```text
┌────────────────────────────────────────┐
│ DevOps Pipeline                         │
│ 构建 / 打包 / 冒烟 / 回归 / 性能 / 报告   │
└───────────────────┬────────────────────┘
                    ↓
┌────────────────────────────────────────┐
│ Python Test Framework                   │
│ 拉起游戏 / 组装用例 / 下发命令 / 采集上传 │
└───────────────────┬────────────────────┘
                    ↓
┌────────────────────────────────────────┐
│ UE C++ Test Plugin                      │
│ TestController / 状态机 / 状态回传 / 性能 │
└───────────────────┬────────────────────┘
                    ↓
┌────────────────────────────────────────┐
│ Observer + Diagnosis + Healer           │
│ 日志截图Trace / 失败归因 / 自修复 / 数据回流 │
└────────────────────────────────────────┘
```

## 5. 实践路线

建议按阶段实现：

```text
阶段 1：JSON DSL + Python mock executor
阶段 2：Markdown Spec → JSON 生成
阶段 3：UE C++ 状态回传 Demo
阶段 4：Python 拉起游戏 + 执行 JSON 用例
阶段 5：DevOps 流水线接入
阶段 6：性能采集与回归检测
阶段 7：Commit Blame 可疑改动定位
阶段 8：失败归因 Diagnoser
阶段 9：Healer 自修复闭环
阶段 10：数据飞轮与模型迭代
```

详见 `10_practice_milestones.md`。

## 6. 最终项目亮点

```text
1. 规格驱动：自然语言 → BDD / Markdown Spec → JSON DSL
2. 三端协作：DevOps 调度 + Python 编排 + UE C++ 执行
3. 状态驱动：通过 UE 状态回传减少 sleep 和不稳定等待
4. 多源观测：日志、截图、UI、Actor、Gameplay State、性能 Trace
5. 智能诊断：失败归因、性能异常解释、可疑 Commit 定位
6. 自修复闭环：JSON Patch、沙箱重跑、人工确认
7. 数据飞轮：生成、执行、失败、修复全链路沉淀训练数据
```

## 7. 一句话总结

> 这是一个以 UE5 游戏自动化测试为场景的 AI-native 测试平台学习项目：LLM 负责测试规划、生成、诊断和修复，Python 负责外部编排，UE C++ 负责游戏内确定性执行和状态回传，DevOps 负责流水线调度和质量门禁。最终目标是把游戏测试从“人工写脚本 + 人工看报告”升级为“规格驱动 + 自动执行 + 智能诊断 + 自修复 + 数据闭环”。
