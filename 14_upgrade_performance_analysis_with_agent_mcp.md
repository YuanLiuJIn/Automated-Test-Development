# 14. UE5 测试 Agent 性能分析模块升级方案

> 基于"Agent + MCP 赋能 UE 性能分析"的设计哲学，将你现有的性能检测模块升级为可交互、可全自动的智能性能分析系统。

## 1. 你当前的能力

```text
已有：
  ✅ 性能指标采集（FPS、FrameTime、Memory、GameThread 等）
  ✅ Z-Score + 环比 + 硬阈值异常检测
  ✅ Commit Blame 可疑改动定位
  ✅ 性能报警（分钟级）

缺失：
  ❌ GPU 帧级分析（哪个 DrawCall 最重、哪个 Shader 是瓶颈）
  ❌ CPU 卡顿根因（Jank 聚类、热路径提取、线程归因）
  ❌ 内存分配溯源（哪个模块/函数分配了什么、为什么）
  ❌ Agent 可交互查询（不能"问"为什么变慢）
  ❌ 全自动报告（不能一键生成完整性能分析报告）
```

## 2. 升级目标

```text
从"指标报警"升级为"根因分析"

当前：FPS 下降 → 报警 → Commit Blame → 人工排查
升级：FPS 下降 → Agent 自动分析 GPU/CPU/内存 → 定位根因 → 输出报告
```

## 3. 整体架构升级

```text
┌─────────────────────────────────────────────────────┐
│ UE5 测试 Agent 性能分析系统                           │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Python 测试框架                                      │
│  ├─ 性能采集（已有）                                   │
│  │   FPS / FrameTime / Memory / GameThread / GPU Time  │
│  ├─ 异常检测（已有）                                   │
│  │   Z-Score / 环比 / 硬阈值                           │
│  ├─ Commit Blame（已有）                               │
│  └─ 性能 MCP 工具（新增）                              │
│      ├─ gpu_analyzer_mcp    → RenderDoc CLI 包装       │
│      ├─ cpu_analyzer_mcp    → UTrace MCP 包装          │
│      └─ memory_analyzer_mcp → Heap2Report 包装         │
│                                                      │
│  Agent 智能分析层（新增）                               │
│  ├─ 交互模式：问答式性能排查                            │
│  │   "这帧为什么卡？" → Agent 自动查 GPU/CPU/内存       │
│  ├─ 全自动模式：批量回归分析                            │
│  │   异常触发 → 自动分析 → 生成 HTML 报告               │
│  └─ 跨会话续传：大型 Trace 分批分析                     │
│      SQLite 持久化进度，对话重启不丢失                   │
│                                                      │
└─────────────────────────────────────────────────────┘
```

## 4. 模块一：GPU 帧分析（新增）

### 设计思路

```text
当性能报警触发时（如 FPS 下降），Agent 自动：
  1. 检查是否有对应版本的 .rdc 抓帧
  2. 如果没有 → 提示"建议在可疑场景录制 RenderDoc 抓帧"
  3. 如果有 → 调用 GPU MCP 工具自动分析
```

### MCP 工具接口

```python
class GPUAnalyzerMCP:
    """GPU 帧分析 MCP 服务"""

    def load_capture(self, rdc_path, mode="dynamic"):
        """加载 .rdc 抓帧文件"""
        pass

    def get_action_tree(self, max_depth=2):
        """获取 GPU 命令树，定位可疑 Pass"""
        pass

    def get_draw_call_detail(self, event_id):
        """获取单个 DrawCall 的完整信息（纹理、Shader、Blend 等）"""
        pass

    def analyze_shader_performance(self, program_id):
        """Mali cycle 分析，识别 ALU/LS/Tex 瓶颈"""
        pass

    def get_frame_screenshot(self, event_id):
        """指定 EID 的渲染输出截图"""
        pass

    def compare_two_frames(self, rdc_path_old, rdc_path_new):
        """对比两个版本的帧，检测 DrawCall 数量、纹理、Shader 变化"""
        pass
```

### 使用场景

```text
场景：CI 性能节点检测到 OpenWorld_001 的 FPS 下降 15%

Agent 自动流程：
  1. 检查 artifacts/OpenWorld_001_v1.2.3.rdc 是否存在
  2. load_capture → 120 个 DrawCall
  3. get_action_tree → MobileBasePass 下 Weapon_HK416 占用异常
  4. get_draw_call_detail(951) → 绑定 10 个 texture，Shader: M_TK_Weapon_Scope
  5. compare_two_frames(v1.2.2.rdc, v1.2.3.rdc) → DrawCall +3，新增 2 个纹理
  6. analyze_shader_performance → fragment LS-bound, 16.1 cycles
  7. 输出："v1.2.3 新增武器皮肤纹理导致 DrawCall 增加，
     Shader LS-bound，建议合并纹理或降低纹理分辨率"
```

## 5. 模块二：CPU 卡顿分析（新增）

### 设计思路

```text
当性能报警触发 P95 FrameTime 异常时，Agent 自动：
  1. 加载对应版本的 .utrace 文件
  2. 运行 prefilter_jank 批量检测所有 Jank 帧
  3. 按热路径签名聚类
  4. 逐 Cluster 分析根因
  5. 生成 Jank 分析报告
```

### MCP 工具接口

```python
class CPUAnalyzerMCP:
    """CPU 卡顿分析 MCP 服务"""

    def load_db(self, db_path):
        """加载 .insights.db"""
        pass

    def get_trace_summary(self):
        """会话元数据 + 线程列表 + 帧统计"""
        pass

    def query_frames(self, min_duration_ms=33):
        """按耗时筛选慢帧"""
        pass

    def get_hotpath(self, frame_index, thread_id):
        """指定帧 + 线程的热路径"""
        pass

    def prefilter_jank(self, jank_method="perfdog"):
        """批量检测所有 Jank 帧 + 聚类"""
        pass

    def get_jank_summary(self):
        """查看聚类结果（按频次排序）"""
        pass

    def start_jank_report(self, max_clusters=20):
        """创建分析计划（支持跨会话续传）"""
        pass

    def save_cluster_analysis(self, cluster_id, root_cause, analysis):
        """持久化每个 Cluster 的分析结果"""
        pass

    def generate_jank_report(self):
        """生成 Markdown + HTML 报告"""
        pass
```

### 使用场景

```text
场景：CI 性能节点检测到 P95 FrameTime 从 22ms 上升到 38ms

Agent 自动流程：
  1. load_db("traces/20260714_perf.insights.db")
  2. prefilter_jank(jank_method="perfdog")
     → 124 帧 Jank 检测，49 帧全 idle 跳过，75 帧分析
     → 33 个 Cluster
  3. get_jank_summary()
     → Cluster #1: 15 帧，GameThread，avg=91ms
       签名：ParallelFor > ParticleSystemComponent (PS_Snow_BigStorm)
     → Cluster #2: 12 帧，RHIThread（GameThread idle 重定向）
  4. 逐 Cluster 分析：
     Cluster #1: 粒子系统 Tick 在 ParallelFor 中序列化执行
     → 建议：限制粒子数量、开启异步 Tick
     Cluster #2: RHIThread 提交过慢
     → 建议：检查 RHI 命令缓冲区大小
  5. generate_jank_report()
     → 输出 jank_report_20260714.html
```

## 6. 模块三：内存分配分析（新增）

### 设计思路

```text
当性能报警触发 Memory 峰值上升时，Agent 自动：
  1. 对比两个版本的内存快照
  2. 定位增量最大的分配点
  3. 追踪完整调用链
  4. 联动源码给出优化建议
```

### MCP 工具接口

```python
class MemoryAnalyzerMCP:
    """内存分析 MCP 服务"""

    def load_file(self, file_path):
        """加载内存快照文件（自动判断 Snapshot/Diff）"""
        pass

    def get_summary(self):
        """总体统计 + Top 5 根节点"""
        pass

    def get_top_allocations(self, n=20, min_size_mb=1.0):
        """全树最大的 N 个分配节点"""
        pass

    def get_call_path(self, node_id):
        """从根到该节点的完整调用链"""
        pass

    def search_function(self, pattern):
        """正则搜索函数名（按大小排序）"""
        pass

    def compare_versions(self, old_snapshot, new_snapshot):
        """两个版本内存 Diff 分析"""
        pass
```

### 使用场景

```text
场景：CI 性能节点检测到 Memory 峰值从 4.2GB 上升到 5.8GB

Agent 自动流程：
  1. compare_versions("v1.2.2_heap.txt", "v1.2.3_heap.txt")
  2. get_summary() → Delta: +1.6GB
  3. get_top_allocations(20, 10.0)
     → CacheMeshDrawCommands: +312 MB
     → LoadObject: +180 MB
     → ParticleSystemComponent::CreateParticles: +95 MB
  4. get_call_path(node_312) → 12 层调用链
  5. search_function("CacheMeshDrawCommands")
     → 在 PrimitiveSceneInfo.cpp:1234
  6. Grep 源码 → 发现新增了大量静态网格体
  7. 输出："v1.2.3 新增关卡导致静态网格体数量增加 40%，
     CacheMeshDrawCommands 内存增长 312 MB。
     建议：检查是否有不必要的网格体加载，或使用 Streaming"
```

## 7. 交互模式

### 问答式性能排查

```text
工程师：OpenWorld_001 这帧 FPS 为什么只有 38？

Agent：
  [load_capture("OpenWorld_001.rdc")]
  → 帧统计：156 DrawCall，其中 MobileBasePass 占 89 个

  [get_action_tree(max_depth=2)]
  → WP_Weapon_HK416: 23,997 verts，耗时 8.2ms
  → PS_Snow_BigStorm: 1,200 particles，耗时 6.5ms

  [analyze_shader_performance("M_TK_Weapon_Scope")]
  → fragment LS-bound, 16.1 cycles
  → 建议：减少 texture fetch，使用简化 Shader

  [get_hotpath(frame_index=5421, thread_id=2)]
  → ParticleSystemComponent::Tick → 45ms → 主要瓶颈

  [search_function("PS_Snow_BigStorm")]
  → 找到粒子系统定义，建议限制 max particles = 200

  综合结论：
  1. GPU：武器 Shader LS-bound（8.2ms）
  2. CPU：粒子系统 Tick 过重（6.5ms）
  3. 优先优化粒子系统，预计 FPS 提升至 52+
```

## 8. 全自动模式

### CI 触发流程

```text
CI 性能节点检测到异常
  ↓
自动触发 Agent 分析：
  ├─ GPU：加载 .rdc → get_action_tree → analyze_shader_performance
  ├─ CPU：加载 .utrace → prefilter_jank → 逐 Cluster 分析
  └─ 内存：加载 heap snapshot → compare_versions → get_top_allocations
  ↓
Agent 自动生成 HTML 报告：
  ├─ 性能回归概览
  ├─ GPU 帧分析（最重 DrawCall + Shader 瓶颈）
  ├─ CPU Jank 分析（聚类结果 + 根因）
  ├─ 内存增量分析（Top 分配点 + 调用链）
  ├─ Commit Blame 关联
  └─ 优化建议优先级
  ↓
报告推送 + 通知相关人员
```

### 跨会话续传

```text
大型 Trace（如 30 分钟游戏录制）可能产生上百个 Jank Cluster：

对话 1：
  prefilter_jank() → 80 个 cluster
  → 分析了 15 个 → Context 快满
  → 告知："已完成 15/80，下轮继续"

对话 2：
  start_jank_report() → 自动检测 15/80 已完成
  → 继续分析 16~30

对话 N：
  generate_jank_report() → 完整报告

所有中间结果持久化到 SQLite，不会丢失。
```

## 9. 大模型"偷懒"问题处理

```text
问题：大模型在分析大量数据时，倾向于深度分析前几个，
      后面的敷衍或跳过。

解决：TODO 表机制
  1. Agent 先扫描所有待分析项，写入 TODO 表
  2. 每分析完一项，标记为 done
  3. 分多次（或并行）运行，每次只处理适量数据
  4. 最终合并所有分析结果

实现：
  TODO 表存储在 SQLite 中
  每次 Agent 启动时检查未完成项
  每次只分配 N 个任务（如 5 个 Cluster）
  完成后标记 → 下次继续
```

## 10. 和现有 Commit Blame 的联动

```text
当前：性能异常 → Commit Blame → 可疑提交列表

升级后：
  性能异常 → Agent 分析 GPU/CPU/内存 → 定位具体根因
  → 结合 Commit Blame 的变更文件列表
  → 输出："FPS 下降 15% 的原因是 ParticleSystem Tick 过重，
    可疑提交 abc123 修改了 ParticleSystemComponent.cpp 的
    MaxParticles 默认值从 200 改为 2000"
```

## 11. 实现优先级

```text
P0（核心，立即做）：
  1. CPU Jank 分析（投入产出比最高）
     你的项目已经有性能指标采集，直接导出 .utrace 即可接入
  2. 全自动报告生成
     异常触发 → 自动分析 → HTML 报告

P1（重要，本周做）：
  3. GPU 帧分析
     需要录制 .rdc 抓帧，CI 自动化稍复杂
  4. 交互式问答
     让工程师能直接问"这帧为什么卡"

P2（增强，下迭代）：
  5. 内存分配分析
     需要采集堆快照，适合版本间回归对比
  6. 跨会话续传
     处理大型 Trace
```

## 12. 一句话总结

> 把你现有的"指标报警"升级为"根因分析"：通过 MCP 把 GPU/CPU/内存分析工具 API 化，让 Agent 能按需查询性能数据、自动定位瓶颈、联动源码给出优化建议。从"FPS 下降了 15%，可能是 abc123 提交导致的"升级为"FPS 下降是因为 ParticleSystem Tick 过重，abc123 提交将 MaxParticles 从 200 改为了 2000，建议回滚或限制粒子数量"。
