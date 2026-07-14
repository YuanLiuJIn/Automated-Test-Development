# 10. 实践里程碑：从 0 到 1 做出可展示项目

> 目标：把完整平台拆成可逐步实现的项目阶段，每个阶段都有明确交付物。

## 阶段 0：准备基础知识

学习内容：

```text
自动化测试基础
UE5 自动化测试基础
Python CLI 与进程管理
JSON Schema
WebSocket / HTTP 通信
基础 DevOps 流水线
LLM Prompt / SFT 基础
```

交付物：

```text
学习笔记
项目架构图
JSON DSL 初版
```

---

## 阶段 1：JSON DSL + Python 本地执行器

目标：先不接 UE，做一个最小测试执行器。

功能：

```text
读取 JSON 用例
逐 step 执行 mock action
记录 step 结果
生成 JSON / HTML 报告
返回 exit code
```

交付物：

```text
cases/sample_case.json
python_test_framework/executor.py
reports/sample_report.html
```

价值：

```text
验证 JSON DSL 和执行框架设计。
```

---

## 阶段 2：BDD / Markdown Spec → JSON 生成

目标：增加测试规格中间层。

功能：

```text
输入 Markdown Spec
LLM 生成 JSON DSL
JSON Schema 校验
失败时输出校验错误
```

交付物：

```text
specs/dungeon_reward.md
cases/dungeon_reward.json
schema/test_case.schema.json
```

价值：

```text
从“一步生成 JSON”升级到“规格驱动测试”。
```

---

## 阶段 3：UE C++ 状态回传 Demo

目标：让 UE 端能返回结构化状态。

功能：

```text
UE 启动本地 TestCommandServer
Python 连接 UE
Python 发送 get_state
UE 返回当前地图、玩家坐标、UI 状态、FPS
```

交付物：

```text
UEAutomationTestPlugin/TestCommandServer
python_test_framework/ue_client.py
state_response.json
```

价值：

```text
打通 Python 和 UE C++ 的通信链路。
```

---

## 阶段 4：Python 拉起游戏 + 执行 JSON 用例

目标：完成最小真实游戏自动化。

功能：

```text
Python 拉起游戏 Build
连接 UE TestController
下发 load_level / move_to / interact / assert_state
采集日志和截图
生成测试报告
```

交付物：

```text
run_test.py
UE TestController
sample UE case 执行报告
```

价值：

```text
项目从模拟执行进入真实 UE 环境。
```

---

## 阶段 5：DevOps 流水线接入

目标：让测试能被流水线自动调用。

功能：

```text
构建节点
冒烟测试节点
性能测试节点
报告上传节点
失败时返回明确 exit code
```

交付物：

```text
pipeline.yml
run_smoke_test.py
run_performance_test.py
artifacts/
```

价值：

```text
自动化测试进入工程流程。
```

---

## 阶段 6：性能采集与回归检测

目标：实现你的核心亮点之一。

功能：

```text
采集 FPS / FrameTime / Memory
保存历史结果
硬阈值判断
环比判断
Z-Score 判断
生成性能回归报告
```

交付物：

```text
performance_metrics.db
baseline_service.py
regression_report.html
```

价值：

```text
从功能测试扩展到性能质量门禁。
```

---

## 阶段 7：Commit Blame 可疑改动定位

目标：把性能报警变成可解释诊断。

功能：

```text
获取当前 commit 与 baseline commit diff
分析变更文件
结合异常指标和测试场景
输出可疑提交排序
```

交付物：

```text
commit_analyzer.py
suspicious_commits.json
llm_diagnosis.md
```

价值：

```text
把报警延迟缩短到分钟级，并给研发可操作线索。
```

---

## 阶段 8：失败归因 Diagnoser

目标：失败后自动判断原因。

功能：

```text
输入 Observation / Log / Screenshot / Spec
输出失败类型
输出证据
输出修复建议
```

交付物：

```text
diagnoser.py
failure_taxonomy.md
diagnosis_report.json
```

价值：

```text
让自动化测试从“报错”升级为“解释为什么错”。
```

---

## 阶段 9：Healer 自修复闭环

目标：实现生成-执行-诊断-修复闭环。

功能：

```text
失败后生成 JSON Patch
沙箱重跑
验证修复是否有效
输出 patch report
人工确认后合入用例库
```

交付物：

```text
healer.py
patches/
sandbox_rerun_report.html
```

价值：

```text
降低自动化用例维护成本。
```

---

## 阶段 10：数据飞轮与模型迭代

目标：把执行数据沉淀为训练数据。

功能：

```text
保存需求 → Spec → JSON → Trace → Diagnosis → Patch
构建 SFT 数据集
构建失败归因评测集
构建好/坏用例偏好数据
```

交付物：

```text
datasets/sft_cases.jsonl
datasets/failure_diagnosis_eval.jsonl
datasets/preference_pairs.jsonl
```

价值：

```text
让系统越用越强。
```

## 最小可展示版本 MVP

如果时间有限，优先做：

```text
1. Markdown Spec → JSON DSL
2. Python 执行器
3. UE 状态回传 Demo
4. 性能采集 + 简单回归检测
5. 报告页面
```

这个版本已经可以很好地展示：

```text
AI 生成测试 + UE 执行 + 性能检测 + 工程闭环
```

## 一句话总结

> 不要一开始就做完整平台。先做 JSON DSL 和 Python 执行器，再打通 UE 状态回传，然后接入流水线和性能检测，最后逐步加入失败诊断、自修复和数据飞轮。这样每一阶段都有可运行、可展示、可复用的交付物。