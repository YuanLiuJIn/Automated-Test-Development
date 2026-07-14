# AI + 自动化测试开发：项目调研与升级路线
当前：
LLM 驱动的 UE5 游戏智能自动化测试框架
设计并实现了一套将大语言模型与 UE5 自动化测试深度融合的智能测试框架。该框架支持自然语言生成
测试用例、操作录制智能转写、JSON 驱动测试执行、以及性能异常自动检测报警四大核心功能。构建 
2300 条游戏测试用例 SFT 数据集，微调 Qwen2.5-7B 模型， 实现自然语言 → 结构化测试用例 JSON 
的自动生成，准确率 92%+。设计基于状态机的可配置测试引擎，支持 16 种操作类型的 JSON 驱动编
排，统一执行 LLM/手工/录制三种来源的测试用例。实现游戏操作录制与 LLM 智能转写系统，通过事件
代理采集角色状态与UI 交互，将冗长操作日志自动转换为标准化测试用例；基于统计异常检测（Z
Score + 环比 + 硬阈值）实现性能回归自动报警，结合 Commit Blame 定位可疑改动，报警延迟缩短至
分钟级。

> 背景：当前目标是把 LLM 与 UE5 游戏自动化测试结合，形成“自然语言生成测试用例、录制转写、JSON 驱动执行、性能异常检测报警、可疑改动定位”的智能测试框架。

## 1. 你的当前方案定位

你目前的项目已经覆盖了 AI 自动化测试里非常关键的四条主线：

```text
1. 自然语言 → 结构化测试用例 JSON
2. 操作录制 → LLM 智能转写 → 标准测试用例
3. JSON 驱动的可配置测试执行引擎
4. 性能回归检测 + Commit Blame 可疑改动定位
```

从行业趋势看，它不是一个“小 demo”，而是接近下一代 AI-native 测试平台的雏形。后续升级重点不应只是继续加 prompt，而是补齐：

```text
测试规格中间层
生成-执行-诊断-自修复闭环
UE/Gauntlet 工程化接入
视觉/状态感知能力
质量门禁与成本治理
模型评测与数据闭环
```

## 2. GitHub 重点项目

### 2.1 Daedalic Test Automation Plugin

地址：`https://github.com/DaedalicEntertainment/ue4-test-automation`

定位：Unreal Engine 集成测试自动化插件，基于 Gauntlet。

可借鉴点：

```text
1. 用 Level 表示 Test Suite
2. 用 Test Actor 表示具体测试
3. Arrange-Act-Assert 测试模型
4. Blueprint 暴露测试能力，降低非 C++ 用户门槛
5. 支持模拟玩家输入、Trigger Box、Delay、Assumption、Skip Reason
6. 支持参数化测试
7. 支持性能预算测试
8. 支持 JUnit XML / 自定义测试报告
9. 通过 Gauntlet / UAT 接入 CI/CD
```

对你的升级启发：

```text
你的 JSON 测试用例可以映射为：
  JSON Case → Test Suite → Test Actor / Test Step

你的 16 种操作类型可以进一步对齐：
  Arrange / Act / Assert / Assume / Cleanup

你的测试执行报告建议输出：
  JUnit XML + JSON Trace + 性能指标 + 截图/录像 + LLM诊断结果
```

---

### 2.2 GauntletAutomationDemo

地址：`https://github.com/S1Lazza/GauntletAutomationDemo`

定位：UE5.2 下 Gauntlet 自动化性能测试 Demo。

可借鉴点：

```text
1. C# Automation Project 控制 Gauntlet 测试
2. C++ GauntletTestController 控制游戏内测试逻辑
3. 使用 UE Performance Report Tool 生成性能报告
4. 使用脚本运行 Windows / Android / VR 设备测试
5. 输出性能 XML / HTML / 图表
6. 支持自定义目标 FPS，如 72 FPS
```

对你的升级启发：

```text
你的性能检测报警可以进一步接入 UE 原生 CsvProfiler / PerfreportTool。

推荐链路：
  JSON Case
    ↓
  Gauntlet Automation C# Driver
    ↓
  UE C++ TestController
    ↓
  CsvProfiler / Unreal Insights / Log
    ↓
  结构化性能指标
    ↓
  Z-Score + 环比 + 阈值 + Commit Blame
```

---

### 2.3 Automated Performance Testing Framework

地址：`https://github.com/Ongun1905/Automated-Performance-Testing-Framework`

定位：UE5 游戏性能测试自动化框架。

可借鉴点：

```text
1. UE 插件负责定义和运行性能测试
2. 控制台工具负责执行、上报和比较
3. MySQL 存储历史性能结果
4. Grafana 展示趋势和对比
5. 当前结果自动与上一次运行结果比较
```

对你的升级启发：

```text
你已有性能异常检测，下一步建议平台化：

UE 测试执行
  ↓
结果采集器
  ↓
指标数据库
  ↓
基线对比 / 异常检测
  ↓
Dashboard
  ↓
报警 + Commit Blame + LLM 诊断摘要
```

推荐指标表：

```text
test_runs:
  run_id, case_id, branch, commit, build_id, platform, map, start_time, status

performance_metrics:
  run_id, avg_fps, p95_frame_time, p99_frame_time, game_thread, render_thread,
  gpu_time, memory_peak, load_time, hitch_count, crash_count

regression_alerts:
  run_id, metric, baseline, current, diff_percent, severity, suspicious_commits
```

---

### 2.4 GETAF: Game Engine Test Automation Framework

地址：`https://github.com/ntufar/getaf`

定位：面向 UE5 3D 游戏的自动化测试框架蓝图。

可借鉴点：

```text
1. 游戏测试金字塔：Unit / Integration / System
2. UE5 Gauntlet + GoogleTest + NUnit + CI/CD
3. XML / JSON 数据驱动测试
4. 性能监控、截图对比、API 测试、数据库测试
5. 统一测试框架层 + 脚本层 + CI 层
```

对你的升级启发：

```text
你的框架可以明确分层：

测试生成层：LLM / SFT / RAG / 录制转写
测试规格层：Markdown Spec / JSON Case / BDD
测试执行层：状态机引擎 / UE Gauntlet / TestController
观测层：日志 / 状态 / 截图 / 性能 / Trace
诊断层：失败归因 / 性能异常 / Commit Blame
治理层：报告 / Dashboard / 数据集回流 / 模型评测
```

---

### 2.5 Playwright Test Agents

地址：`https://playwright.dev/docs/test-agents`

定位：Web 自动化测试中的 planner / generator / healer 三 Agent 工作流。

核心模式：

```text
需求 / PRD
  ↓
planner：生成测试计划 Markdown
  ↓
generator：生成可执行测试代码
  ↓
healer：运行失败后自动修复测试
```

对你的升级启发非常大：

```text
不要直接从自然语言生成 JSON。
建议增加“测试规格中间层”：

自然语言需求
  ↓
测试点 / 风险点
  ↓
BDD / Markdown 测试规格
  ↓
结构化 JSON 用例
  ↓
UE 状态机执行
```

这样可以显著提升：

```text
可审查性
可追溯性
数据集质量
模型微调质量
失败诊断质量
```

---

### 2.6 AI Testing Agent

地址：`https://github.com/davidodidi/ai-testing-agent`

定位：Claude + Playwright 的 Agentic 测试框架。

可借鉴点：

```text
1. 自然语言目标 → JSON Plan
2. Playwright 执行
3. selector 失败后提取 DOM 快照
4. LLM 生成新 selector
5. 修复成功后缓存 selector 映射
6. AI 视觉断言
7. HTML + JSON 报告
```

对你的 UE5 场景启发：

```text
Web 的 DOM 快照可以类比 UE 的：
  UI Widget Tree
  Actor Tree
  Gameplay State
  Input Event Trace
  Screenshot / Frame Buffer

Web 的 selector healing 可以类比 UE 的：
  UI 元素定位修复
  Actor 查找策略修复
  操作参数修复
  等待条件修复
```

---

### 2.7 Midscene.js

地址：`https://github.com/web-infra-dev/midscene`

定位：视觉驱动 UI 自动化，支持自然语言步骤、视觉定位、自然语言断言。

可借鉴点：

```text
1. 用自然语言描述 UI 操作
2. 多模态模型通过截图定位元素
3. 支持视觉断言
4. 可接入 Playwright / Vitest
5. 也支持 Agent 自主测试
```

对你的 UE5 场景启发：

```text
UE 游戏 UI 和 3D 场景很多时候没有稳定 DOM。
因此可以引入“视觉定位 + 状态机执行”的双通道：

优先：事件代理 / UI树 / Actor状态 精确执行
兜底：截图 + 多模态模型 视觉定位
```

## 3. 内部知识库重点文章

> 下面是值得重点阅读的内部实践文章，建议按主题分组看，而不是逐篇孤立看。

### 3.1 AI 自动化测试平台类

文章：`663809` 金融系统 AI 自动化测试

可借鉴点：

```text
1. 接口测试无人值守闭环
2. UI 跨端自然语言执行
3. 无侵入 Mock Filter 隔离复杂依赖
4. 代码剪枝降低 token 成本
5. 分级治理：轻量模型处理原子任务
6. 增量复用：只对变更代码执行 AI
7. PICT 正交覆盖 + 白盒路径补齐
8. 异常捕捉 - 自动诊断 - 问题自愈闭环
```

对你的升级启发：

```text
你现在有性能异常报警，建议补“测试环境隔离”和“成本治理”：

环境隔离：Mock / Replay / Sandbox
成本治理：代码剪枝 / 上下文裁剪 / 增量生成 / 小模型执行
覆盖治理：PICT 组合测试 + 白盒路径补齐 + LLM 场景扩展
```

---

### 3.2 AI 原生 UI 自动化平台类

文章：`663767` UI 自动化测试平台从产品设计到 AI 协同落地

可借鉴点：

```text
1. 两段式用例生成：先测试点，再测试步骤
2. Web / Skill / CLI / OpenAPI 多形态产品矩阵
3. UnifiedAgent 统一执行抽象
4. 可插拔执行引擎：视觉引擎 + 控件树引擎
5. 可测性工程：环境作为一等公民
6. 大模型规划 + 小模型执行
7. 失败归因和用例自愈规划
8. 探索 - 沉淀 - 回归 飞轮
```

对你的升级启发：

```text
建议你的平台也拆成两段式：

自然语言需求
  ↓
测试点列表：测什么
  ↓ 人工/规则确认
测试步骤 JSON：怎么测
  ↓
状态机执行
```

这比直接端到端生成 JSON 更稳定。

---

### 3.3 测试用例生成 Pipeline 类

文章：`660041` 从需求到测试用例设计：AI 辅助测试用例自动生成

可借鉴点：

```text
1. 不建议一步到位生成完整用例
2. 使用 Pipeline-based 生成
3. BDD 拆解需求
4. 生成测试方案
5. 多维 Review
6. 扩写测试方案
7. 生成结构化用例
8. 用步骤树表达分支路径
```

对你的项目高度相关：

```text
你的 SFT 数据集可以升级为多阶段数据：

原始需求
  → BDD
  → 测试点
  → 扩写方案
  → JSON Case
  → 执行 Trace
  → 修复后 JSON
```

这样比单纯 `自然语言 → JSON` 的 SFT 更强。

---

### 3.4 生成-跑测-自修复闭环类

文章：`655739` 自动化测试「生成-跑测-自修复」全闭环实践

可借鉴点：

```text
1. 用例生成
2. 平台跑测
3. 定时轮询状态
4. 失败日志 + 截图分析
5. 自动修复用例
6. 多轮收敛
7. 失败原因统计
```

对你的升级启发：

```text
你已有 JSON 执行和性能报警，建议加：

失败用例
  ↓
收集日志 / 截图 / Actor状态 / UI状态 / Trace
  ↓
LLM 失败归因
  ↓
生成修复建议：
    - 修改等待条件
    - 修改元素定位
    - 修改操作参数
    - 增加前置状态
    - 标记产品缺陷
  ↓
沙箱重跑
  ↓
形成 patch / 人工确认
```

---

### 3.5 LLM + 树搜索生成测试代码类

文章：`655626` AutoHarness 实验复盘：用 LLM + 树搜索自动生成测试代码

可借鉴点：

```text
1. LLM 生成测试代码
2. 编译/运行反馈驱动迭代
3. Harness Tree 管理候选版本
4. Thompson Sampling 平衡探索和利用
5. Critic 分析错误，Refiner 修复代码
6. 合法动作率作为搜索目标
```

对你的项目启发：

```text
你可以把“JSON 用例”也做成搜索对象：

候选 JSON Case
  ↓
执行成功率 / 覆盖率 / 性能异常检测能力 / 稳定性评分
  ↓
LLM 变异：修改步骤、增加断言、调整等待、补充前置条件
  ↓
选择更优 Case
```

这会让你的框架从“生成一次”升级为“自动优化测试用例”。

---

### 3.6 GUI Agent 视觉能力类

文章：`652015` Florence-2 模型在 GUI Agent 中的微调实践

可借鉴点：

```text
1. UI 图标识别需要输出可控语义，而不是自由描述
2. 构建核心图标、通用图标、负样本三类数据
3. 用 LoRA 微调视觉语言模型
4. 用负样本压低误报
5. 灰度、回滚、失败样本回流
```

对 UE5 游戏测试启发：

```text
游戏 UI/图标/技能按钮/状态图标 很适合做垂域视觉微调。

推荐数据集：
  技能图标 → skill_xxx
  背包图标 → inventory
  地图按钮 → map
  任务图标 → quest
  异常状态 → debuff_xxx
  负样本 → 非交互图标 / 装饰元素 / 背景纹理
```

## 4. 你的框架升级蓝图

### 4.1 当前版本

```text
自然语言
  ↓
微调模型生成 JSON
  ↓
状态机执行 16 种操作
  ↓
性能检测报警
  ↓
Commit Blame
```

### 4.2 建议升级为 V2 架构

```text
需求 / 录制 / 历史缺陷 / 玩法目标
  ↓
Planner：生成测试点、风险点、BDD
  ↓
Reviewer：覆盖率审查、边界补齐、PICT 组合
  ↓
Generator：生成结构化 JSON Case
  ↓
Validator：Schema 校验、可执行性校验、资源校验
  ↓
Executor：UE Gauntlet + 状态机 + TestController
  ↓
Observer：日志、截图、Actor状态、UI树、性能指标、Trace
  ↓
Diagnoser：失败归因、性能异常解释、可疑 Commit 定位
  ↓
Healer：用例自修复、等待条件修复、定位修复、参数修复
  ↓
Curator：沉淀为高质量 SFT / DPO / 评测数据
```

### 4.3 建议新增的关键模块

```text
1. Test Spec 中间层
   Markdown / BDD / YAML，便于人工审核和模型学习。

2. Test Case Schema Registry
   管理 JSON 用例 schema、操作类型、参数约束、版本迁移。

3. UE State Observer
   采集 Actor、UI、Gameplay Tag、任务状态、坐标、相机、血量、背包等结构化状态。

4. Visual Grounding Adapter
   截图 + 图标/控件识别，作为事件代理失效时的兜底。

5. Failure Diagnosis Agent
   输入日志、截图、Trace、状态、性能曲线，输出失败类型和修复建议。

6. Self-Healing Sandbox
   修复后的 JSON 先在沙箱重跑，不直接污染主库。

7. Performance Baseline Service
   管理每个地图/平台/画质/测试场景的性能基线。

8. Coverage Evaluator
   评估用例覆盖：地图、玩法、UI、任务、操作类型、状态转移、性能场景。

9. Dataset Curator
   自动沉淀：需求→测试点→JSON→执行轨迹→失败归因→修复版本。
```

## 5. 你最值得对标的三个方向

### 方向一：UE 游戏自动化工程底座

对标：

```text
Daedalic Test Automation Plugin
GauntletAutomationDemo
Automated Performance Testing Framework
```

你要补齐：

```text
Gauntlet/UAT 标准化执行
UE TestController
JUnit/HTML/JSON 报告
性能报告工具接入
CI/CD 质量门禁
```

### 方向二：AI 测试生成与自修复闭环

对标：

```text
Playwright Test Agents
AI Testing Agent
内部生成-跑测-自修复实践
```

你要补齐：

```text
Planner / Generator / Healer 分工
测试规格中间层
失败诊断与自动修复
修复 patch 审核
多轮收敛机制
```

### 方向三：游戏 GUI / 场景视觉智能

对标：

```text
Midscene.js
GUI Agent 视觉模型微调实践
YOLO 浮层态检测实践
```

你要补齐：

```text
UE UI 图标数据集
控件/状态/浮层识别
视觉断言
截图语义诊断
视觉模型微调或 LoRA
```

## 6. 推荐阅读顺序

```text
第一组：UE 工程底座
  1. Daedalic Test Automation Plugin
  2. GauntletAutomationDemo
  3. Automated Performance Testing Framework

第二组：AI 用例生成
  4. 从需求到测试用例设计：Pipeline-based 生成
  5. Playwright Test Agents
  6. AutoHarness 实验复盘

第三组：闭环与自修复
  7. 生成-跑测-自修复闭环实践
  8. AI Testing Agent
  9. AI 原生 UI 自动化平台设计

第四组：视觉与游戏 UI
  10. Midscene.js
  11. Florence-2 GUI 图标识别微调
  12. YOLO 浮层态检测实践
```

## 7. 你的项目简历表述可升级方向

当前表述已经不错，但可以升级成更“大厂范式”的说法：

```text
设计并实现 AI-native UE5 游戏自动化测试平台，构建“测试规格生成—JSON 用例编排—Gauntlet 状态机执行—多源观测—失败诊断—性能回归报警—用例自修复”的闭环体系。平台支持自然语言/录制/手工三类用例来源，基于 SFT 模型将玩法目标转化为结构化测试 DSL，并通过 UE 事件代理采集 Actor、UI、Gameplay State 与性能指标，实现测试执行、异常检测和可疑提交定位的自动化。进一步引入 Planner/Generator/Healer 多 Agent 架构，将测试设计拆解为 BDD 规格、覆盖审查、JSON 生成、沙箱执行和失败自修复多个阶段，提升用例可审计性、执行稳定性和数据闭环质量。
```

## 8. 下一步建议

建议你优先做三件事：

```text
1. 给当前 JSON 用例增加 Spec 中间层
   自然语言 → BDD/测试点 → JSON，而不是一步到位。

2. 接入 Gauntlet 标准执行链路
   把你的状态机执行器包装成 UE TestController / UAT 可调用的测试任务。

3. 增加失败归因与自修复模块
   每次失败都输出：产品缺陷 / 环境问题 / 用例问题 / 定位问题 / 等待问题 / 性能回归。
```

如果这三件做完，你的项目会从“LLM 生成测试用例”升级为“AI-native 游戏自动化测试平台”。