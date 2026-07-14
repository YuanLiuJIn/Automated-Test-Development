# 04. Python / UE C++ / DevOps 三端协作架构

> 目标：把 UE 游戏自动化测试拆成三个工程端：Python 测试框架、UE C++ 游戏端、DevOps 流水线。

## 1. 总体关系

```text
DevOps 流水线
  负责：构建、打包、分阶段调度、质量门禁
      ↓
Python 测试框架
  负责：拉起游戏、组装用例、下发命令、采集产物、上传报告
      ↓
UE C++ 测试插件
  负责：执行游戏内动作、观察状态、回传事件、采集性能
```

一句话：

```text
DevOps 是调度器，Python 是编排器，UE C++ 是游戏内执行器。
```

## 2. DevOps 流水线层

DevOps 不负责具体测试逻辑，它负责把测试流程放进工程流水线。

典型节点：

```text
1. 拉代码
2. 编译 UE 工程
3. 打包游戏 Build
4. 准备测试环境
5. 冒烟测试
6. 功能回归
7. 性能测试
8. 稳定性测试
9. 汇总报告
10. 质量门禁 / 报警
```

## 3. DevOps 传给 Python 的参数

不同流水线节点可以传不同入参：

```text
--build-path          游戏包路径
--case-suite          用例集合：smoke/regression/performance/soak
--platform            Win64 / Android / iOS / Console
--map                 测试地图
--graphics-quality    画质档位
--collect             采集指标列表
--upload-cloud        是否上传云端
--baseline-id         性能基线
--branch              当前分支
--commit              当前 Commit
--run-mode            运行模式
--retry               失败重试次数
```

示例：

```bash
python run_test.py \
  --build-path D:/Builds/Game-Win64 \
  --case-suite performance \
  --platform Win64 \
  --map Dungeon_001 \
  --graphics-quality High \
  --collect fps,frametime,memory,log,screenshot,trace \
  --upload-cloud true \
  --baseline-id main_latest \
  --commit abc123
```

## 4. Python 测试框架层

Python 是平台中枢。

职责：

```text
读取 JSON 用例
根据平台和流水线参数组装测试任务
拉起游戏进程
拉起测试工具
连接 UE C++ 插件
下发测试步骤
等待状态回传
采集日志、截图、性能、Trace
重命名和归档产物
上传云端
生成报告
返回 exit code 给 DevOps
```

推荐模块：

```text
python_test_framework/
├─ cli.py                 # 命令行入口
├─ config_loader.py        # 读取流水线参数
├─ case_loader.py          # 读取 JSON 用例
├─ case_assembler.py       # 平台/地图/画质用例组装
├─ game_launcher.py        # 拉起游戏
├─ tool_launcher.py        # 拉起采集工具
├─ ue_client.py            # 和 UE 通信
├─ executor.py             # 执行主循环
├─ artifact_manager.py     # 产物命名与归档
├─ metrics_collector.py    # 性能数据采集
├─ uploader.py             # 上传云端
├─ reporter.py             # 报告生成
└─ exit_code.py            # 返回流水线状态码
```

## 5. Python 与 UE 的通信协议

可以选：

```text
WebSocket：适合双向实时通信
HTTP：简单，适合请求/响应
TCP Socket：控制力强
文件轮询：简单但低效
日志解析：兜底，不建议作为主协议
UE Remote Control：适合特定工具场景
```

推荐：

```text
控制命令：WebSocket / TCP
状态查询：WebSocket / HTTP
大文件产物：文件系统 / 对象存储
```

## 6. UE C++ 测试插件层

UE C++ 端负责游戏内部能力。

推荐模块：

```text
UEAutomationTestPlugin/
├─ TestCommandServer       # 接收 Python 命令
├─ TestCommandDispatcher   # 命令分发
├─ TestActionExecutor      # 执行动作
├─ GameStateObserver       # 游戏状态观察
├─ UIStateObserver         # UI 状态观察
├─ ActorStateObserver      # Actor 状态观察
├─ PerformanceCollector    # 性能指标采集
├─ ScreenshotService       # 截图
├─ TraceService            # Trace / CSV / Insights
├─ TestEventBus            # 事件回传
└─ TestController          # 测试生命周期管理
```

## 7. UE 返回给 Python 的状态格式

不要只返回字符串日志，应该返回结构化 JSON。

```json
{
  "type": "state_update",
  "timestamp": 123456.7,
  "game_state": {
    "level": "Dungeon_001",
    "phase": "Combat",
    "quest_state": "InProgress"
  },
  "player_state": {
    "location": [120.5, 88.2, 0],
    "health": 85,
    "is_dead": false
  },
  "ui_state": {
    "visible_widgets": ["QuestPanel", "HealthBar", "SkillBar"],
    "active_dialog": null
  },
  "perf": {
    "fps": 58.2,
    "frame_time_ms": 17.1,
    "memory_mb": 4200
  }
}
```

## 8. 三端执行完整例子

性能测试节点：

```text
DevOps 传参：case_suite=performance, map=OpenWorld_001
  ↓
Python 拉起游戏，连接 UE
  ↓
Python 下发 load_level
  ↓
UE 加载地图，返回 LevelLoaded
  ↓
Python 下发 start_perf_capture
  ↓
UE 开启性能采集
  ↓
Python 下发 run_flight_path
  ↓
UE 控制角色或相机按固定路线移动
  ↓
UE 持续回传性能指标
  ↓
Python 收集 csv/trace/log/screenshot
  ↓
Python 计算性能回归
  ↓
DevOps 发布报告并决定是否通过
```

## 9. Exit Code 设计

Python 最终要返回明确退出码给流水线：

```text
0 = 通过
1 = 功能失败
2 = 性能回归
3 = 环境失败
4 = 崩溃
5 = 用例无效
6 = 工具异常
```

这样 DevOps 可以精确判断下一步动作。

## 10. 一句话总结

> UE 自动化测试的三端架构中，DevOps 负责“什么时候跑、跑哪一段”，Python 负责“怎么拉起、怎么编排、怎么采集、怎么报告”，UE C++ 负责“游戏内部怎么执行、怎么判断状态、怎么回传结果”。三者边界清晰，平台才能稳定扩展。