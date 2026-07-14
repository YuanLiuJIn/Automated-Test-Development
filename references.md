# references

> 本页整理本项目后续学习和实践时可以参考的方向。具体资料可围绕这些关键词继续扩展。

## 1. UE / 游戏自动化测试

```text
Unreal Engine Automation Test Framework
Unreal Engine Functional Testing
Unreal Engine Gauntlet Automation Framework
Unreal Automation Tool UAT
Unreal Insights
CSV Profiler
Performance Report Tool
```

建议重点理解：

```text
1. UE 自动化测试体系有哪些层级
2. Gauntlet 如何启动游戏并管理测试生命周期
3. TestController 如何在游戏内执行测试逻辑
4. 性能数据如何采集和导出
```

## 2. 游戏测试工程项目

可参考公开项目方向：

```text
Daedalic Test Automation Plugin
GauntletAutomationDemo
Automated Performance Testing Framework
Game Engine Test Automation Framework
```

关注点：

```text
测试套件组织方式
Gauntlet 集成方式
性能报告生成方式
CI/CD 集成方式
```

## 3. AI 自动化测试

关键词：

```text
AI test generation
LLM test case generation
self-healing test automation
AI visual assertion
agentic testing
Playwright Test Agents
Midscene.js
```

重点关注：

```text
planner / generator / healer 分工
测试规格中间层
失败后自修复
视觉定位与语义断言
```

## 4. 测试用例生成与 BDD

关键词：

```text
Behavior Driven Development
Gherkin Given When Then
test case generation pipeline
BDD to test automation
JSON DSL test framework
```

重点关注：

```text
如何从需求生成测试点
如何从 BDD 生成可执行用例
如何设计 JSON Schema
如何做测试覆盖审查
```

## 5. 性能回归检测

关键词：

```text
performance regression testing
statistical anomaly detection
Z-Score anomaly detection
software performance baseline
commit blame performance regression
```

重点关注：

```text
性能基线如何建立
不同平台和地图如何分组比较
如何结合阈值、环比、统计异常
如何把异常关联到代码变更
```

## 6. 模型训练与数据闭环

关键词：

```text
SFT test case generation
DPO preference optimization
LLM evaluator
failure diagnosis dataset
self-healing test dataset
```

重点关注：

```text
需求到测试点的数据
Spec 到 JSON 的数据
失败到诊断的数据
诊断到修复 Patch 的数据
好用例和坏用例的偏好数据
```

## 7. 推荐实践顺序

```text
1. JSON DSL + Python mock executor
2. Markdown Spec → JSON Generator
3. Python ↔ UE 状态通信 Demo
4. UE TestController 执行真实步骤
5. 性能采集与回归检测
6. DevOps 流水线接入
7. 失败归因与自修复
8. 数据集沉淀与模型微调
```