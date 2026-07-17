# 13. Agent 设计模式在 UE5 测试 Agent 中的落地方案

> 基于"Agent = 上下文 × 工具 × 循环"的设计框架，将六条核心改进点逐一落地到你的 UE5 自动化测试 Agent。

## 改进点一：上下文分区

### 当前问题

你的 System Prompt 可能是一大段混合文本，包含规则、Action 列表、当前状态等，每次请求全量发送，Cache 利用率低。

### 改进方案

将 Prompt 拆成五层：

```text
┌─────────────────────────────────────────────┐
│ Zone 1：系统区（完全固定，100% Cache）       │
├─────────────────────────────────────────────┤
│ 角色：UE5 游戏自动化测试 Agent               │
│ 规则：                                       │
│   - 必须使用 JSON DSL 输出测试步骤           │
│   - 必须先规划测试点，再生成 JSON             │
│   - 每个 step 必须包含 timeout               │
│   - 禁止跳过核心断言                          │
│   - 性能指标必须指定阈值                      │
│ JSON Schema 定义（固定不变）                  │
│ 约 3-5K tokens                               │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Zone 2：工具区（半固定）                      │
├─────────────────────────────────────────────┤
│ Action 类型摘要（16 种操作分类）：            │
│   环境类：load_level, reset_world            │
│   角色类：move_to, interact, attack          │
│   UI 类：click_ui, assert_ui_visible         │
│   状态类：wait_state, assert_quest_state     │
│   性能类：start_perf_capture                 │
│ 约 0.5K tokens（摘要模式）                    │
│                                               │
│ 按需展开详情：                                 │
│   load_level：加载指定地图                     │
│     - level: string (required)，地图名称       │
│     - timeout_sec: int (default 60)           │
│     - 示例：{"action":"load_level",            │
│        "params":{"level":"Dungeon_001"}}       │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Zone 3：环境区（准静态，会话内不变）          │
├─────────────────────────────────────────────┤
│ 当前平台：Win64                               │
│ 画质档位：High                                │
│ 可用地图列表：                                 │
│   - Dungeon_001 (副本)                        │
│   - OpenWorld_001 (开放世界)                   │
│   - MainCity (主城)                           │
│ 可用 Actor 列表：                              │
│   - NPC_QuestGiver, DungeonEntrance, ...      │
│ 约 2-3K tokens                                │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Zone 4：记忆区（长期持久）                     │
├─────────────────────────────────────────────┤
│ 用户偏好：                                    │
│   - 常用地图：Dungeon_001                     │
│   - 常用测试类型：功能回归 + 性能检测          │
│                                               │
│ 项目知识：                                    │
│   - Boss 死亡后需等待 3 秒结算 UI 才出现       │
│   - RewardChest 的可交互距离为 150            │
│   - 背包满时奖励通过邮件发送                   │
│                                               │
│ 历史失败模式：                                 │
│   - "RewardChest not interactable"            │
│     → 原因：距离过远，修复：增加 move_to 步骤   │
│ 约 1-2K tokens（相关性注入）                   │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Zone 5：对话区（高度动态）                     │
├─────────────────────────────────────────────┤
│ 当前任务需求                                  │
│ 已生成的测试点 / BDD                          │
│ 历史步骤 + 执行结果（压缩后）                  │
│ 约 10-30K tokens                              │
└─────────────────────────────────────────────┘
```

### 实现要点

```python
class ContextBuilder:
    def __init__(self):
        self.system_zone = self.load_system_prompt()   # 固定，读文件
        self.tool_zone_summary = self.load_tool_summary() # 固定
        self.env_zone = None  # 会话初始化时设置一次

    def build_context(self, task, memory_results, history):
        """按固定顺序组装上下文，最大化 Cache 命中"""

        # Zone 1: 完全固定（Cache 友好）
        context = [self.system_zone]

        # Zone 2: 工具摘要（固定）
        context.append(self.tool_zone_summary)

        # Zone 3: 环境（会话内稳定）
        if self.env_zone is None:
            self.env_zone = self.collect_environment()
        context.append(self.env_zone)

        # Zone 4: 记忆（相关性注入）
        context.append(self.format_memory(memory_results))

        # Zone 5: 对话（动态，放在最后）
        context.append(task)
        context.append(history)

        return context
```

**关键原则**：把稳定的放前面，动态的放后面。Zone 1-2 完全不变 → 高 Cache 命中。

---

## 改进点二：渐进式 Action 发现

### 当前问题

16 种操作类型的完整 Schema 全部塞进 Prompt，大部分场景只用到其中 4-5 种。

### 改进方案

```text
第一层：Action 分类摘要（始终在上下文）
  环境类：load_level, reset_world, set_graphics_quality
  角色类：move_to, interact, attack, use_skill
  UI 类：click_ui, assert_ui_visible, wait_ui_state
  状态类：wait_state, assert_quest_state, assert_inventory
  性能类：start_perf_capture, stop_perf_capture

  约 0.3K tokens

第二层：按需展开详情
  当 Planner 确定测试场景后，只加载该场景需要的 Action 详情
  例如：副本测试 → 加载 load_level, move_to, interact,
        wait_state, assert_quest_state, start_perf_capture 的详情

  约 1-2K tokens（而非全部 16 种的 5K+ tokens）
```

### 实现

```python
ACTION_REGISTRY = {
    "load_level": {
        "category": "环境类",
        "summary": "加载指定地图",
        "detail": """...完整参数说明...""",
        "required_scenarios": ["功能测试", "性能测试"]
    },
    "move_to": {
        "category": "角色类",
        "summary": "移动角色到目标位置",
        "detail": """...完整参数说明...""",
        "required_scenarios": ["功能测试"]
    },
    # ...
}

def build_action_context(test_scenario):
    """根据测试场景按需加载 Action 详情"""
    summary = format_action_summary(ACTION_REGISTRY)
    needed = get_actions_for_scenario(test_scenario)
    details = [ACTION_REGISTRY[a]["detail"] for a in needed]
    return summary + "\n" + "\n".join(details)
```

---

## 改进点三：智能失败反馈

### 当前问题

失败时只返回"Step 6 failed: RewardChest not interactable"，LLM 不知道原因和修复方向。

### 改进方案

四层反馈结构：

```json
{
  "step_id": 6,
  "status": "failed",

  "Level 1：错误事实": {
    "action": "interact",
    "target": "RewardChest",
    "error": "Target not interactable"
  },

  "Level 2：详细分析": {
    "target_exists": true,
    "target_interactable": false,
    "player_distance": 320,
    "required_distance": 150,
    "diagnosis": "玩家距离目标过远，不在可交互范围内"
  },

  "Level 3：修复建议": [
    {
      "type": "insert_step_before",
      "before_step_id": 6,
      "suggested_step": {
        "action": "move_to",
        "params": {
          "target": "RewardChest",
          "distance_lte": 120
        }
      },
      "confidence": 0.91
    },
    {
      "type": "alternative",
      "description": "修改 interact 的 distance 参数为 350",
      "confidence": 0.72
    }
  ],

  "Level 4：辅助信息": {
    "nearby_actors": ["DungeonGate", "HealthPickup"],
    "player_state": {"health": 85, "location": [100, 200, 0]},
    "ui_state": {"visible_widgets": ["QuestPanel", "HealthBar"]},
    "recent_successful_steps": [
      "Step 4: move_to DungeonEntrance succeeded (distance=80)",
      "Step 5: interact DungeonEntrance succeeded (distance=80)"
    ]
  }
}
```

### 实现

```python
class FailureFeedbackBuilder:
    def build(self, step, observation):
        feedback = {}

        # Level 1：错误事实
        feedback["error_facts"] = {
            "action": step.action,
            "target": step.params.get("target"),
            "error": observation.error
        }

        # Level 2：详细分析
        feedback["analysis"] = self.diagnose(step, observation)

        # Level 3：修复建议
        feedback["suggestions"] = self.generate_fixes(
            step, observation, self.memory.get_similar_failures()
        )

        # Level 4：辅助信息
        feedback["context"] = {
            "nearby_actors": observation.nearby_actors,
            "player_state": observation.player_state,
            "ui_state": observation.ui_state,
            "recent_steps": observation.recent_successful_steps
        }

        return feedback

    def diagnose(self, step, observation):
        """诊断失败原因"""
        if step.action == "interact":
            if observation.target_interactable == False:
                if observation.player_distance > observation.required_distance:
                    return "玩家距离目标过远，不在可交互范围内"
                else:
                    return "目标存在但不可交互，可能被锁定或未激活"
        # ... 其他 action 的诊断规则
```

---

## 改进点四：循环检测升级

### 当前问题

你已经有"同屏同操作 ≥3 次"的检测，但还可以更强。

### 改进方案

三层循环检测：

```python
class LoopDetector:
    def __init__(self):
        self.recent_actions = []  # 最近 N 个操作
        self.action_sequences = {}  # 操作序列模式计数

    def check(self, current_step):
        """三层循环检测"""

        # 层 1：同操作重复检测（你已有的）
        if self.detect_same_action_loop(current_step):
            return self.build_warning(
                "检测到同一操作重复执行",
                severity="medium",
                suggestion="尝试不同方法或检查前置条件"
            )

        # 层 2：操作序列模式检测
        if self.detect_sequence_pattern():
            return self.build_warning(
                "检测到操作序列循环",
                severity="high",
                suggestion="当前策略可能无效，建议切换策略或终止"
            )

        # 层 3：无进展检测
        if self.detect_no_progress():
            return self.build_warning(
                "操作无实际进展",
                severity="critical",
                suggestion="任务可能已陷入死锁，建议终止并报告"
            )

        return None  # 无循环

    def detect_sequence_pattern(self):
        """检测操作序列是否在重复"""
        if len(self.recent_actions) < 10:
            return False

        # 检查最近 10 个操作是否形成重复模式
        seq = self.recent_actions[-10:]
        mid = len(seq) // 2
        if seq[:mid] == seq[mid:]:
            return True
        return False

    def detect_no_progress(self):
        """检测是否连续多步无任何状态变化"""
        if len(self.recent_actions) < 5:
            return False

        recent_states = [a.get("state_snapshot") for a in self.recent_actions[-5:]]
        # 如果最近 5 步的状态完全一样 → 无进展
        return len(set(str(s) for s in recent_states)) == 1

    def build_warning(self, message, severity, suggestion):
        return {
            "type": "loop_warning",
            "message": message,
            "severity": severity,
            "suggestion": suggestion,
            "recent_actions": self.recent_actions[-5:]
        }
```

---

## 改进点五：Memory 跨会话复用

### 当前问题

每次测试都是"从零开始"，不知道上次教过什么、上次失败怎么修的。

### 改进方案

```python
class TestMemory:
    """跨会话的测试记忆系统"""

    def __init__(self, memory_file=".aitest/TEST_MEMORY.md"):
        self.memory_file = memory_file

    def remember_failure(self, error_pattern, root_cause, fix):
        """记住一次失败和修复方案"""
        entry = {
            "error_pattern": error_pattern,
            "root_cause": root_cause,
            "fix": fix,
            "timestamp": datetime.now(),
            "case_id": current_case_id
        }
        self.save(entry)

    def find_similar_failures(self, current_error):
        """查找历史上类似的失败"""
        history = self.load()
        similar = []

        for entry in history:
            if self.error_similar(current_error, entry["error_pattern"]):
                similar.append(entry)

        return similar[:3]  # 返回最相似的 3 条

    def remember_quirk(self, quirk_type, description, workaround):
        """记住项目的特殊行为（quirk）"""
        entry = {
            "type": quirk_type,
            "description": description,
            "workaround": workaround,
            "timestamp": datetime.now()
        }
        self.save(entry)

    def get_quirks_for_map(self, map_name):
        """获取特定地图的已知特殊行为"""
        return [q for q in self.load() if q.get("map") == map_name]
```

### 记忆文件结构（TEST_MEMORY.md）

```markdown
# 测试记忆

## 失败模式

### RewardChest 距离过远
- 错误：RewardChest not interactable
- 根因：用例缺少移动到宝箱前的步骤
- 修复：在 interact 前插入 move_to(target=RewardChest, distance_lte=120)
- 出现次数：3
- 最近出现：2026-07-14

### Boss 死亡后结算 UI 延迟
- 错误：SettlementPanel not visible within 2s
- 根因：Boss 死亡动画 + 结算触发有延迟
- 修复：wait_state 的 timeout 从 2s 改为 5s
- 出现次数：5
- 最近出现：2026-07-13

## 项目特殊行为

### Dungeon_001 宝箱
- 宝箱在 Boss 死亡后 1.5s 才变为可交互
- 建议在 Boss 死亡后增加 wait_state(condition="RewardChest interactable")

### 背包满时奖励
- 背包满时，副本奖励通过邮件发送
- 断言时不要检查背包，改为检查邮件
```

### 注入方式

```python
def build_context(self, task, map_name):
    context = []

    # ... Zone 1-3 ...

    # Zone 4：注入相关记忆
    failures = self.memory.find_similar_failures(task.description)
    quirks = self.memory.get_quirks_for_map(map_name)

    memory_context = f"""
    ## 历史相关失败（参考，避免重复）：
    {self.format_failures(failures)}

    ## 当前地图已知行为：
    {self.format_quirks(quirks)}
    """

    context.append(memory_context)
    return context
```

---

## 改进点六：结果智能裁剪

### 当前问题

UE 日志可能有几千行，全量返回会撑爆上下文。

### 改进方案

```python
class ResultTrimmer:
    def __init__(self, head_lines=10, tail_lines=10, max_total=200):
        self.head_lines = head_lines
        self.tail_lines = tail_lines
        self.max_total = max_total

    def trim_log(self, log_text):
        """裁剪 UE 日志"""
        lines = log_text.split("\n")
        total = len(lines)

        # 日志短，不裁剪
        if total <= self.max_total:
            return log_text

        # 保留开头 + 结尾 + 所有 Error/Warning
        head = lines[:self.head_lines]
        tail = lines[-self.tail_lines:]

        # 提取所有错误和警告（即使它们在中间）
        errors = [l for l in lines[self.head_lines:-self.tail_lines]
                  if "Error" in l or "Warning" in l or "Fatal" in l or "Crash" in l]

        result = head.copy()
        if errors:
            result.append(f"\n... [省略 {total - self.head_lines - self.tail_lines} 行，"
                          f"其中 {len(errors)} 条 Error/Warning] ...\n")
            result.extend(errors)
        else:
            result.append(f"\n... [省略 {total - self.head_lines - self.tail_lines} 行] ...\n")
        result.extend(tail)

        return "\n".join(result)

    def trim_tool_output(self, output, tool_type):
        """根据工具类型选择裁剪策略"""
        if tool_type == "execute_command":
            return self.trim_log(output)
        elif tool_type == "screenshot":
            # 截图不裁剪文本，只裁剪图片大小
            return output
        elif tool_type == "dump_actor_tree":
            # Actor 树保留关键 Actor
            return self.extract_key_actors(output)
        else:
            return output
```

---

## 完整架构升级对比

### 当前架构

```text
自然语言需求
  ↓
SFT 模型 → JSON Case
  ↓
Python 执行器 → UE TestController
  ↓
简单日志 + 性能报警
```

### 升级后架构

```text
自然语言需求
  ↓
┌─────────────────────────────┐
│ Agent 上下文引擎              │
│ ├─ Zone 1: 系统规则（Cache） │
│ ├─ Zone 2: Action 渐进加载   │
│ ├─ Zone 3: 当前环境          │
│ ├─ Zone 4: Memory 注入       │
│ └─ Zone 5: 对话历史（压缩）   │
└─────────────────────────────┘
  ↓
Planner → Reviewer → Generator
  ↓
JSON Case
  ↓
┌─────────────────────────────┐
│ 执行引擎                     │
│ ├─ 步进式执行                 │
│ ├─ 三层循环检测               │
│ ├─ 智能终止                   │
│ └─ 状态追踪（三级 ID）        │
└─────────────────────────────┘
  ↓
┌─────────────────────────────┐
│ 观测与反馈                    │
│ ├─ 四层失败反馈               │
│ ├─ 结果智能裁剪               │
│ ├─ Memory 更新               │
│ └─ 自动压缩历史               │
└─────────────────────────────┘
```

---

## 实现优先级

```text
P0（立即做，投入产出比最高）：
  1. 上下文分区（5 层）
  2. 智能失败反馈（4 层结构）
  3. 结果智能裁剪

P1（本周做，显著提升稳定性）：
  4. Memory 跨会话复用
  5. 循环检测升级（3 层）

P2（下个迭代做，锦上添花）：
  6. 渐进式 Action 发现
```

## 一句话总结

> 把 Agent 设计模式落到 UE5 测试 Agent 上，核心是六件事：上下文分区让 Cache 高效，渐进式 Action 减少 Token 浪费，四层失败反馈让 AI 能自己诊断和修复，三层循环检测避免死循环，Memory 跨会话让 Agent 越用越聪明，结果裁剪防止上下文被日志撑爆。
