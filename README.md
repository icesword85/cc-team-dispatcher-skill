# CC-Team Dispatcher Skill

> 让 OpenClaw Agent 通过 tmux 调用 CC-Team（4 个不同 AI 模型的 Claude Code 实例）协同工作。

## 这是什么？

CC-Team Dispatcher 是一个 [OpenClaw](https://openclaw.ai) Agent 技能（Skill），让体系内的任何 Agent 都能通过 tmux 命令调用 **CC-Team**——一个由 4 个不同 AI 模型驱动的 Claude Code 协作团队。

简单说：**它是 Agent 和 CC-Team 之间的桥梁。**

## CC-Team 是什么？

CC-Team 是一个**多模型 AI 编程协作系统**：

| 窗格 | 模型 | 定位 | 相对成本 |
|------|------|------|----------|
| 0 | **Qwen 3.7 Plus**（阿里云） | 日常编码、格式化、简单任务 | 1x |
| 1 | **Doubao-Seed-2.1-Turbo**（火山引擎） | 任务拆解、项目协调、中等复杂度 | ~2x |
| 2 | **GLM-5.2**（火山引擎） | 架构设计、模块划分、技术方案 | ~4x |
| 3 | **DeepSeek V4 Pro**（火山引擎） | 复杂调试、深度代码审查、疑难问题 | ~6x |

4 个实例通过 tmux 管理在同一个会话中，共享项目目录，各自独立运行。

## 解决了什么问题？

**痛点：**
- Agent 想让 CC-Team 帮忙执行任务，但不知道怎么正确调用
- `claude -p "..."` 命令行方式有认证问题
- 需要标准化的调用协议，降低其他 Agent 的使用门槛

**这个 Skill 的作用：**
- ✅ 标准化调用流程，任何 Agent 看一遍就能用
- ✅ 明确"什么任务该给哪个模型"
- ✅ 记录最佳实践，避免重复踩坑

## 快速开始

### 1. 启动 CC-Team

```bash
# tmux 模式（推荐）
start-cc-team --tmux

# 指定项目目录
start-cc-team --tmux --dir /path/to/project

# 检查状态
start-cc-team --status
```

### 2. 发送任务

```bash
# 向 Qwen（日常编码）发送任务
tmux send-keys -t cc-team:0.0 "格式化 src/utils.js 并修复 lint 错误" Enter

# 向 GLM（架构设计）发送任务
tmux send-keys -t cc-team:0.2 "设计一个新的用户认证模块的接口" Enter

# 向 DeepSeek（深度分析）发送任务
tmux send-keys -t cc-team:0.3 "分析这段代码的内存泄漏原因" Enter
```

### 3. 读取输出

```bash
# 读取窗格 0 的最近 50 行输出
tmux capture-pane -t cc-team:0.0 -p -S -50
```

### 4. 停止

```bash
start-cc-team --stop
```

## 模型选择指南

| 场景 | 推荐模型 | 原因 |
|------|----------|------|
| 代码格式化、简单修复 | 窗格 0（Qwen） | 便宜快速 |
| 功能开发、任务拆解 | 窗格 1（Doubao） | 上下文理解好 |
| 系统设计、技术方案 | 窗格 2（GLM） | 架构能力强 |
| 复杂 bug、性能优化 | 窗格 3（DeepSeek） | 深度分析能力 |
| 完整代码审查流水线 | 0 → 2 → 3 → 1 | 从浅到深再到整理 |

## 多模型协作模式

### 流水线模式

按顺序传递任务，逐步深化：

```bash
# 1. Qwen 做初步代码检查和格式化
tmux send-keys -t cc-team:0.0 "检查代码规范并格式化" Enter
sleep 30

# 2. GLM 做架构审查
tmux send-keys -t cc-team:0.2 "审查刚才的代码架构是否合理" Enter
sleep 30

# 3. DeepSeek 做性能分析
tmux send-keys -t cc-team:0.3 "分析这段代码的性能瓶颈" Enter
sleep 30

# 4. Doubao 整理报告
tmux send-keys -t cc-team:0.1 "整理前面三个模型的输出，生成总结报告" Enter
```

### 并行模式

同时给多个模型发不同任务：

```bash
tmux send-keys -t cc-team:0.0 "任务 A：重构 utils 目录" Enter
tmux send-keys -t cc-team:0.2 "任务 B：设计新的 API 接口" Enter
tmux send-keys -t cc-team:0.3 "任务 C：分析内存泄漏问题" Enter

# 定期检查各窗格进度
tmux capture-pane -t cc-team:0.0 -p -S -20
tmux capture-pane -t cc-team:0.2 -p -S -20
tmux capture-pane -t cc-team:0.3 -p -S -20
```

## 文件位置

| 文件 | 说明 |
|------|------|
| `~/.claude/scripts/start-cc-team.sh` | 一键启动脚本 |
| `~/.claude/scripts/cc-launch.sh` | 单实例启动器 |
| `/opt/homebrew/bin/start-cc-team` | 全局软链接 |

## 注意事项

1. **不要并发调用同一个窗格** — 等待当前任务完成后再发送新任务
2. **任务描述要清晰** — CC-Team 需要明确的输入才能给出好的输出
3. **监控执行进度** — 定期用 `tmux capture-pane` 检查输出
4. **项目目录共享** — 所有窗格共享同一个项目目录，注意文件冲突
5. **资源占用** — 4 个 Claude Code 实例会占用 2-4GB 内存

## 技术细节

- **依赖**：tmux、Claude Code、start-cc-team 脚本
- **适用平台**：macOS
- **格式**：纯 Markdown（SKILL.md）

## License

MIT
