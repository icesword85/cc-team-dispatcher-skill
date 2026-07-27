# CC-Team Dispatcher Skill

## 用途
通过 tmux 调用 CC-Team 的 4 个 Claude Code 实例执行编程、调试、架构设计等任务。

## CC-Team 配置

### 模型分配
| 窗格 | 模型 | 适用场景 |
|------|------|----------|
| 0 | Qwen 3.7 Plus | 日常编码、格式化、简单任务 |
| 1 | Doubao-Seed-2.1-Turbo | 任务拆解、项目协调、中等复杂度 |
| 2 | GLM-5.2 | 架构设计、模块划分、技术方案 |
| 3 | DeepSeek V4 Pro | 复杂调试、深度代码审查、疑难问题 |

### 文件位置
- 启动脚本：`~/.claude/scripts/start-cc-team.sh`
- 单实例启动：`~/.claude/scripts/cc-launch.sh`
- 全局命令：`start-cc-team`

## 操作步骤

### 1. 启动 CC-Team
```bash
# tmux 模式（推荐）
start-cc-team --tmux

# 或指定项目目录
start-cc-team --tmux --dir /path/to/project

# 检查状态
start-cc-team --status
```

### 2. 向指定窗格发送任务（必须验证）

**⚠️ 关键规则：发送后必须验证，确认命令已执行才能离开**

```bash
# 第一步：发送任务
tmux send-keys -t cc-team:0.0 "你的任务描述" Enter

# 第二步：等待 2 秒让 Claude Code 处理
sleep 2

# 第三步：验证命令是否真的执行了
tmux capture-pane -t cc-team:0.0 -p -S -30
```

**验证标准（必须满足以下至少一条）：**
- ✅ 看到 "Thought for Xs" — 模型已开始思考
- ✅ 看到 "✻ Worked/Crunched/Brewed/Baked" — 任务已执行
- ✅ 看到模型输出内容（代码、分析、回答等）

**如果验证失败（出现以下情况）：**
- ❌ 命令文本停留在 `❯` 后面，没有处理迹象
- ❌ 没有任何 "Thought" 或执行标记

**补发回车强制提交：**
```bash
tmux send-keys -t cc-team:0.0 Enter
sleep 2
tmux capture-pane -t cc-team:0.0 -p -S -30  # 再次验证
```

**完整示例（向 4 个窗格发任务并验证）：**
```bash
# 窗格 0
tmux send-keys -t cc-team:0.0 "任务 A" Enter
sleep 2
tmux capture-pane -t cc-team:0.0 -p -S -30 | grep -q "Thought\|Worked\|Crunched" && echo "✅ 窗格 0 已执行" || tmux send-keys -t cc-team:0.0 Enter

# 窗格 1
tmux send-keys -t cc-team:0.1 "任务 B" Enter
sleep 2
tmux capture-pane -t cc-team:0.1 -p -S -30 | grep -q "Thought\|Worked\|Crunched" && echo "✅ 窗格 1 已执行" || tmux send-keys -t cc-team:0.1 Enter

# 窗格 2
tmux send-keys -t cc-team:0.2 "任务 C" Enter
sleep 2
tmux capture-pane -t cc-team:0.2 -p -S -30 | grep -q "Thought\|Worked\|Crunched" && echo "✅ 窗格 2 已执行" || tmux send-keys -t cc-team:0.2 Enter

# 窗格 3
tmux send-keys -t cc-team:0.3 "任务 D" Enter
sleep 2
tmux capture-pane -t cc-team:0.3 -p -S -30 | grep -q "Thought\|Worked\|Crunched" && echo "✅ 窗格 3 已执行" || tmux send-keys -t cc-team:0.3 Enter
```

### 3. 读取任务输出
```bash
# 读取窗格 0 的最近 50 行输出
tmux capture-pane -t cc-team:0.0 -p -S -50

# 读取窗格 1 的最近 50 行输出
tmux capture-pane -t cc-team:0.1 -p -S -50

# 读取窗格 2 的最近 50 行输出
tmux capture-pane -t cc-team:0.2 -p -S -50

# 读取窗格 3 的最近 50 行输出
tmux capture-pane -t cc-team:0.3 -p -S -50
```

### 4. 检查任务状态
```bash
# 列出所有 tmux 会话
tmux ls

# 列出 cc-team 的所有窗格
tmux list-panes -t cc-team -F "#{pane_index}: #{pane_current_command}"
```

### 5. 停止 CC-Team
```bash
# 使用脚本停止
start-cc-team --stop

# 或手动停止
tmux kill-session -t cc-team
```

## 选择模型指南

### 选择窗格 0（Qwen 3.7 Plus）当：
- 代码格式化、简单重构
- 快速修复语法错误
- 生成模板代码
- 日常编码任务

### 选择窗格 1（Doubao-Seed-2.1-Turbo）当：
- 需要任务拆解和规划
- 多步骤项目协调
- 中等复杂度的功能开发
- 需要较好的上下文理解

### 选择窗格 2（GLM-5.2）当：
- 系统架构设计
- 模块划分和接口设计
- 技术方案评审
- 设计模式应用

### 选择窗格 3（DeepSeek V4 Pro）当：
- 复杂的 bug 调试
- 性能优化分析
- 深度代码审查
- 疑难问题排查
- 安全漏洞分析

## 多模型协作模式

### 流水线模式

**⚠️ 每一步都必须验证上一命令已执行后再进入下一步**

```bash
# 1. Qwen 做初步代码检查
tmux send-keys -t cc-team:0.0 "检查这个文件的代码规范并格式化" Enter
sleep 2
tmux capture-pane -t cc-team:0.0 -p -S -30 | grep -q "Thought\|Worked" || tmux send-keys -t cc-team:0.0 Enter

# 等待完成后（看到模型输出），GLM 做架构审查
sleep 30
tmux send-keys -t cc-team:0.2 "审查刚才的代码架构是否合理" Enter
sleep 2
tmux capture-pane -t cc-team:0.2 -p -S -30 | grep -q "Thought\|Worked" || tmux send-keys -t cc-team:0.2 Enter

# 等待完成后，DeepSeek 做深度分析
sleep 30
tmux send-keys -t cc-team:0.3 "分析这段代码的性能瓶颈" Enter
sleep 2
tmux capture-pane -t cc-team:0.3 -p -S -30 | grep -q "Thought\|Worked" || tmux send-keys -t cc-team:0.3 Enter

# 等待完成后，Doubao 整理报告
sleep 30
tmux send-keys -t cc-team:0.1 "整理前面三个模型的输出，生成总结报告" Enter
sleep 2
tmux capture-pane -t cc-team:0.1 -p -S -30 | grep -q "Thought\|Worked" || tmux send-keys -t cc-team:0.1 Enter
```

### 并行模式

**⚠️ 每个 send-keys 后都必须验证，不要连发不验**

```bash
# 向多个模型发送不同任务（每个都要验证！）

# 窗格 0
tmux send-keys -t cc-team:0.0 "任务 A：重构 utils 目录" Enter
sleep 2
tmux capture-pane -t cc-team:0.0 -p -S -30 | grep -q "Thought\|Worked" || tmux send-keys -t cc-team:0.0 Enter

# 窗格 2
tmux send-keys -t cc-team:0.2 "任务 B：设计新的 API 接口" Enter
sleep 2
tmux capture-pane -t cc-team:0.2 -p -S -30 | grep -q "Thought\|Worked" || tmux send-keys -t cc-team:0.2 Enter

# 窗格 3
tmux send-keys -t cc-team:0.3 "任务 C：分析内存泄漏问题" Enter
sleep 2
tmux capture-pane -t cc-team:0.3 -p -S -30 | grep -q "Thought\|Worked" || tmux send-keys -t cc-team:0.3 Enter
```

## 注意事项

1. **不要并发调用同一个窗格**：等待当前任务完成后再发送新任务
2. **任务描述要清晰**：CC-Team 需要明确的输入才能给出好的输出
3. **监控执行进度**：定期用 `tmux capture-pane` 检查输出
4. **项目目录共享**：所有窗格共享同一个项目目录，注意文件冲突
5. **资源占用**：4 个 Claude Code 实例会占用 2-4GB 内存

## 常见问题

### Q: 如何切换项目目录？
A: 停止当前 CC-Team，然后用 `--dir` 参数重新启动：
```bash
start-cc-team --stop
start-cc-team --tmux --dir /new/project/path
```

### Q: 任务卡住了怎么办？
A: 按 `Escape` 中断当前任务，然后重新发送任务描述。

### Q: 如何查看某个窗格的完整历史？
A: 使用更大的 `-S` 参数：
```bash
tmux capture-pane -t cc-team:0.0 -p -S -500
```

### Q: 如何让 CC-Team 读取某个文件？
A: 在任务描述中明确指定文件路径：
```bash
tmux send-keys -t cc-team:0.0 "读取 src/main.js 并分析代码结构" Enter
```
