# 08. 性能回归检测与 Commit Blame

> 目标：把“性能报警”升级为“性能异常解释 + 可疑改动定位”。

## 1. 什么是性能回归？

性能回归指：

```text
新版本比历史基线明显变慢、变卡、变耗内存。
```

游戏中常见性能回归：

```text
FPS 下降
FrameTime 上升
Hitch 增多
内存峰值上升
地图加载变慢
GPU 时间上升
GameThread 时间上升
崩溃率增加
```

## 2. 为什么游戏性能回归难？

因为性能受很多因素影响：

```text
硬件环境
画质设置
地图资源
玩家路线
怪物数量
特效数量
网络状态
后台进程
编译配置
```

所以必须固定测试条件：

```text
固定机器
固定平台
固定地图
固定路线
固定画质
固定测试时长
固定采样指标
```

## 3. 推荐采集指标

```text
avg_fps
min_fps
p95_frame_time
p99_frame_time
game_thread_ms
render_thread_ms
gpu_time_ms
memory_peak_mb
hitch_count
load_time_ms
actor_count
draw_calls
crash_count
```

## 4. 三类异常检测

你已有的组合是合理的：

```text
Z-Score + 环比 + 硬阈值
```

### 4.1 硬阈值

例如：

```text
avg_fps < 45 → 失败
p95_frame_time > 33ms → 失败
memory_peak > 6000MB → 失败
```

适合明确性能预算。

### 4.2 环比

比较当前运行和上一次运行：

```text
FPS 从 60 降到 52，下降 13.3%
```

适合快速发现近期退化。

### 4.3 Z-Score

比较当前值和历史分布：

```text
z = (current - mean) / std
```

如果：

```text
|z| > 3
```

说明偏离历史正常范围较大。

## 5. 为什么三者要结合？

单独使用有问题：

```text
硬阈值：不能发现“还没低于阈值但明显变差”的情况
环比：容易受上一次波动影响
Z-Score：需要足够历史数据
```

组合后更稳：

```text
硬阈值负责底线
环比负责近期变化
Z-Score 负责统计异常
```

## 6. 性能基线服务

建议建立 Performance Baseline Service。

基线维度：

```text
platform
map
graphics_quality
case_id
branch
hardware_profile
build_config
```

不要用一个全局基线比较所有场景。

## 7. 性能异常报告

```json
{
  "case_id": "openworld_flight_path_001",
  "metric": "p95_frame_time",
  "baseline": 24.5,
  "current": 38.2,
  "diff_percent": 55.9,
  "z_score": 4.1,
  "severity": "critical",
  "related_metrics": {
    "game_thread_ms": "+48%",
    "gpu_time_ms": "+2%",
    "actor_count": "+210%"
  }
}
```

## 8. 从报警到归因

只报警不够，要解释：

```text
是哪类性能变差？
CPU 还是 GPU？
哪个场景？
从哪个步骤开始？
可能与哪些代码改动有关？
```

例子：

```text
P95 FrameTime 明显上升
GameThread 上升，GPU 无明显变化
ActorCount 增加 3 倍
最近改动 MonsterSpawner.cpp
```

诊断：

```text
疑似怪物生成逻辑导致 Actor 数量异常增长，引发 GameThread 压力上升。
```

## 9. Commit Blame 输入

```text
当前失败 run
baseline run
当前 commit
baseline commit
变更文件列表
代码 diff
模块归属
性能异常指标
失败步骤
历史缺陷
```

## 10. Commit Blame 排名因子

```text
变更文件与失败模块相关度
变更文件与性能指标相关度
提交时间距离
历史相似问题
代码 diff 语义
调用链关系
测试用例覆盖关系
```

输出：

```json
{
  "suspicious_commits": [
    {
      "commit": "abc123",
      "files": ["Source/Game/MonsterSpawner.cpp"],
      "reason": "修改了怪物生成数量逻辑，与 ActorCount 和 GameThread 异常相关",
      "score": 0.87
    }
  ]
}
```

## 11. LLM 在性能诊断中的作用

LLM 适合：

```text
综合多指标解释
阅读代码 diff
生成可读诊断报告
根据历史问题寻找相似模式
给出排查建议
```

不适合直接决定：

```text
是否阻断流水线
是否一定是某个提交的问题
是否删除报警
```

最终门禁仍应由规则和统计结果决定。

## 12. 一句话总结

> 性能回归检测要从“指标超阈值报警”升级为“场景化基线 + 多指标异常检测 + 代码变更关联 + 可疑提交排序”。Z-Score、环比和硬阈值负责发现异常，Commit Blame 和 LLM 诊断负责解释异常。