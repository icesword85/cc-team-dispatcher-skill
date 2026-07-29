---
name: cc-team-dispatcher
description: "通过 tmux 调用 CC-Team（4 个不同 AI 模型的 Claude Code 实例）协同工作。当用户需要多模型协作编程、任务分发、架构审查、复杂调试时使用。"
---

# CC-Team Dispatcher Skill

通过 tmux 可靠地向 4 个 Claude Code 实例分发任务，带状态检测、多行安全发送、接收验证和重试机制。

## CC-Team 配置

### 模型分配

| 窗格 | 模型 | 适用场景 |
|------|------|----------|
| 0 | Qwen 3.7 Plus | 日常编码、格式化、简单任务 |
| 1 | Doubao-Seed-2.1-Turbo | 任务拆解、项目协调、中等复杂度 |
| 2 | GLM-5.2 | 架构设计、模块划分、技术方案 |
| 3 | DeepSeek V4 Pro | 复杂调试、深度代码审查、疑难问题 |

### 会话约定

- tmux session 名：`cc-team`
- 窗格格式：`cc-team:0.{0..3}`
- 空闲提示符特征：行尾出现 `❯` 或 `$`，且 2 秒内无新输出

### 文件位置

- 启动脚本：`~/.claude/scripts/start-cc-team.sh`
- 单实例启动：`~/.claude/scripts/cc-launch.sh`
- 全局命令：`start-cc-team`

---

## ⚠️ 关键规则：Claude Code 多行输入机制

**这是最容易踩的坑，必须先理解再操作。**

Claude Code 的终端输入框中：
- **`Enter`** = 提交当前输入（发送给模型）
- **`C-j`（Ctrl+J）** = 在当前输入中插入换行（多行输入）

### 问题根因

当用 `tmux send-keys -t cc-team:0.2 "第一行\n第二行\n第三行" Enter` 发送多行任务时：

1. 文本中的 `\n` 被 Claude Code 检测为粘贴（bracketed paste），整段文本进入输入框
2. 但紧跟在文本后的 `Enter` 被粘贴检测机制吞掉，或因时序问题未注册为提交
3. **结果：任务文本停留在输入框，没有被提交执行**

### 正确做法

根据任务复杂度选择发送方式：

#### 方式 A：单行短任务（最简单）

```bash
tmux send-keys -t cc-team:0.0 "你的单行任务描述" Enter
```

#### 方式 B：多行任务 - 文本与 Enter 分开发送（推荐）

```bash
# 第一步：发送任务文本（不含 Enter）
tmux send-keys -t cc-team:0.2 "第一行内容
第二行内容
第三行内容"

# 第二步：等待 0.5 秒让 Claude Code 处理完粘贴
sleep 0.5

# 第三步：单独发送 Enter 提交
tmux send-keys -t cc-team:0.2 Enter
```

#### 方式 C：超长/复杂任务 - 临时文件法（最可靠）

```bash
# 1. 将任务写入临时文件
cat > /tmp/cc-task-glm.md << 'TASK_EOF'
# 任务：重构认证模块

## 背景
当前 auth.ts 有 500 行，职责混杂。

## 需求
1. 拆分为 token.ts / session.ts / middleware.ts
2. 保持对外接口不变
3. 补充单元测试

## 约束
- 不要引入新依赖
- 保持 TypeScript 严格模式通过
TASK_EOF

# 2. 发送单行命令让模型读取文件
tmux send-keys -t cc-team:0.2 "请阅读 /tmp/cc-task-glm.md 中的任务说明并执行" Enter
```

**选择建议：**
- 1 行简单指令 -> 方式 A
- 2~15 行中等任务 -> 方式 B（文本和 Enter 分两步发）
- 15 行以上或含代码/表格/特殊字符 -> 方式 C（临时文件法）

---

## 核心操作流程

### 0. 前置检查（必做）

发送任何指令前，先确认 session 存在：

```bash
tmux has-session -t cc-team 2>/dev/null && echo "OK" || echo "NO_SESSION"
```

如果 session 不存在，先启动：

```bash
start-cc-team --tmux --dir /path/to/project
```

等待所有窗格初始化完成（约 10~20 秒），再继续。

---

### 1. 检测窗格是否空闲（发送前必做）

原理：连续 2 次捕获窗格内容，比较末尾 5 行是否不变 + 末行含提示符。

```bash
cc_is_ready() {
  local pane=$1
  local snap1=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -10 | tail -5)
  sleep 1
  local snap2=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -10 | tail -5)
  if [ "$snap1" = "$snap2" ] && echo "$snap2" | tail -1 | grep -qE '[❯>$] *$'; then
    return 0
  else
    return 1
  fi
}

cc_wait_ready() {
  local pane=$1
  local i=0
  while [ $i -lt 60 ]; do
    if cc_is_ready "$pane"; then return 0; fi
    sleep 2
    i=$((i+2))
  done
  return 1
}
```

---

### 2. 安全发送指令

**核心原则：多行任务必须将文本发送和 Enter 提交分开执行。**

#### 单行任务

```bash
tmux send-keys -t cc-team:0.0 "格式化 src/utils.ts" Enter
```

#### 多行任务（文本 + 延迟 + Enter）

```bash
# 发送文本（注意：引号内可以包含换行）
tmux send-keys -t cc-team:0.2 "请重构 src/auth.ts：
1. 拆分为 token.ts 和 session.ts
2. 保持对外接口不变
3. 补充单元测试"

# 等待粘贴被处理
sleep 0.5

# 单独提交
tmux send-keys -t cc-team:0.2 Enter
```

#### 含特殊字符的任务（buffer + 分离 Enter）

当任务含 `$`、反引号、`"` 等 shell 特殊字符时，用 buffer 粘贴避免解析问题：

```bash
# 1. 写入临时文件
TMPFILE=$(mktemp /tmp/cc-task-XXXXXX.txt)
cat > "$TMPFILE" << 'ENDOFTASK'
请重构 src/utils/auth.ts，要求：
1. 把 verifyToken() 拆成 3 个函数
2. 错误信息用 "error_${code}" 格式
3. 确保 $TOKEN 环境变量不会被打印
ENDOFTASK

# 2. 加载到 buffer 并粘贴
tmux load-buffer -b cc-task-buf "$TMPFILE"
rm -f "$TMPFILE"
tmux send-keys -t cc-team:0.0 C-c    # 清掉可能的残留输入
sleep 0.3
tmux paste-buffer -b cc-task-buf -t cc-team:0.0

# 3. 等待粘贴完成，再单独发 Enter
sleep 0.5
tmux send-keys -t cc-team:0.0 Enter

# 4. 清理
tmux delete-buffer -b cc-task-buf 2>/dev/null
```

> **为什么 buffer + 分离 Enter？**
> - paste-buffer 粘贴文本时，Claude Code 检测到粘贴会将整段文本放入输入框
> - 但粘贴完成后的 Enter 可能被粘贴检测吞掉
> - 分两步发（先粘贴，等 0.5s，再 Enter）确保提交成功

---

### 3. 验证指令是否被成功接收（发送后必做）

发送后 2~3 秒，检查窗格是否出现处理迹象：

```bash
cc_verify() {
  local pane=$1
  sleep 3
  local tail
  tail=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -30)
  
  # 检查是否有执行标记
  if echo "$tail" | grep -qE "Thought for|✻ |⏺ |Bash\(|Read\(|Searched|Working"; then
    echo "✅ 窗格 $pane 已开始处理"
    return 0
  fi
  
  # 检查任务文本是否还停留在输入框（未被提交）
  if echo "$tail" | tail -5 | grep -qE '❯.*[^\s]'; then
    echo "⚠️ 窗格 $pane 任务可能未提交，文本在输入框中"
    return 1
  fi
  
  echo "⚠️ 窗格 $pane 状态不明"
  return 1
}
```

**验证通过标准（满足任一即可）：**
- 看到 `Thought for Xs` - 模型已开始思考
- 看到 `✻ Worked/Crunched/Brewed/Baked` - 任务已执行
- 看到 `⏺` 或工具调用（`Bash(`、`Read(` 等）- 模型正在操作
- 任务文本已从 `❯` 输入行消失

**验证失败时的补发：**

```bash
# 如果任务停留在输入框，补发 Enter
tmux send-keys -t cc-team:0.2 Enter
sleep 3
tmux capture-pane -t cc-team:0.2 -p -S -30
```

---

### 4. 完整发送流程（含重试）

```bash
cc_dispatch() {
  local pane=$1
  local task_text=$2       # 任务文本（可含换行）
  local keyword=$3         # 任务中独特的开头关键词，用于验证
  local max_retries=${4:-3}

  # 等待空闲
  echo "[pane $pane] 等待空闲..."
  local i=0
  while [ $i -lt 60 ]; do
    local s1=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -10 | tail -5)
    sleep 1
    local s2=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -10 | tail -5)
    if [ "$s1" = "$s2" ] && echo "$s2" | tail -1 | grep -qE '[❯>$] *$'; then
      break
    fi
    sleep 1
    i=$((i+2))
  done

  echo "[pane $pane] 空闲，准备发送"

  local attempt=1
  while [ $attempt -le $max_retries ]; do
    echo "[pane $pane] 第 $attempt 次尝试发送..."

    # 发送文本（不含 Enter）
    tmux send-keys -t "cc-team:0.$pane" "$task_text"
    sleep 0.5
    # 单独发送 Enter
    tmux send-keys -t "cc-team:0.$pane" Enter

    # 验证
    sleep 3
    local tail
    tail=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -30)
    if echo "$tail" | grep -qE "Thought for|✻ |⏺ |Bash\(|Read\("; then
      echo "[pane $pane] ✅ 发送成功"
      return 0
    fi

    echo "[pane $pane] 第 $attempt 次未验证到，重试..."
    tmux send-keys -t "cc-team:0.$pane" C-c
    sleep 1
    attempt=$((attempt+1))
  done

  echo "[pane $pane] ❌ 发送失败，已重试 $max_retries 次"
  return 1
}
```

---

### 5. 读取任务输出

```bash
# 读取最近 100 行
tmux capture-pane -t cc-team:0.0 -p -S -100

# 读取更多历史
tmux capture-pane -t cc-team:0.2 -p -S -500
```

等待任务完成的轮询函数：

```bash
cc_wait_done() {
  local pane=$1
  local timeout=${2:-300}  # 默认超时 5 分钟
  local waited=0
  local last_lines=""

  while [ $waited -lt $timeout ]; do
    sleep 5
    waited=$((waited+5))
    local current
    current=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -10 | tail -5)
    if [ "$current" = "$last_lines" ]; then
      if echo "$current" | tail -1 | grep -qE '[❯>$] *$'; then
        return 0  # 完成
      fi
    fi
    last_lines="$current"
  done
  return 1  # 超时
}
```

---

### 6. 停止 CC-Team

```bash
start-cc-team --stop
# 或直接：
tmux kill-session -t cc-team
```

---

## 多模型协作模式

### 流水线模式

每个步骤都必须验证上一命令已执行后再进入下一步：

```bash
# 1. Qwen 做初步代码检查
tmux send-keys -t cc-team:0.0 "检查 src/main.js 的代码规范并格式化" Enter
sleep 3
tmux capture-pane -t cc-team:0.0 -p -S -30 | grep -qE "Thought|✻" || tmux send-keys -t cc-team:0.0 Enter

# 等待完成后（轮询或手动检查），GLM 做架构审查
# ... 确认窗格 0 完成后 ...
tmux send-keys -t cc-team:0.2 "审查刚才的代码架构是否合理" Enter
sleep 3
tmux capture-pane -t cc-team:0.2 -p -S -30 | grep -qE "Thought|✻" || tmux send-keys -t cc-team:0.2 Enter

# 以此类推...
```

### 并行模式

同时给多个窗格发不同任务，每个都要验证：

```bash
# 窗格 0
tmux send-keys -t cc-team:0.0 "任务 A：重构 utils 目录" Enter
sleep 3
tmux capture-pane -t cc-team:0.0 -p -S -30 | grep -qE "Thought|✻" || tmux send-keys -t cc-team:0.0 Enter

# 窗格 2
tmux send-keys -t cc-team:0.2 "任务 B：设计新的 API 接口" Enter
sleep 3
tmux capture-pane -t cc-team:0.2 -p -S -30 | grep -qE "Thought|✻" || tmux send-keys -t cc-team:0.2 Enter
```

---

## 最佳实践

1. **多行任务必须分离 Enter**：先发文本，sleep 0.5s，再单独发 Enter
2. **发送前等空闲**：这是最容易被忽略但最关键的一步
3. **发送后必验证**：检查是否出现 `Thought` / `✻` / `⏺` 等执行标记
4. **复杂任务用临时文件**：超过 15 行的任务写文件，发单行命令让模型读取
5. **多窗格并发时**，每个窗格独立重试，不互相阻塞
6. **长任务定期检查**：用 `capture-pane` 检查进度，不要傻等
7. **幂等性**：设计任务时尽量让它可重复执行，方便重试

## 常见故障排查

| 现象 | 可能原因 | 解决 |
|------|---------|------|
| 任务文本在输入框但不执行 | 多行粘贴后 Enter 被吞 | 单独发 `tmux send-keys -t cc-team:0.X Enter` |
| 只有第一行被提交 | 换行符被当作 Enter | 用方式 B（分离 Enter）或方式 C（临时文件） |
| 指令没反应 | CC 还在思考，指令插到中间了 | 发前等空闲（cc_wait_ready） |
| 出现奇怪输出 | 特殊字符被 shell 解析 | 用 buffer + heredoc 方式 |
| 验证总失败 | keyword 太普通或等待时间不够 | 增加等待时间到 5s，换独特关键词 |
| session 不存在 | CC-Team 没启动或崩了 | `start-cc-team --tmux` 重启 |

## 常见问题

### Q: 如何切换项目目录？
A: 停止当前 CC-Team，然后用 `--dir` 参数重新启动：
```bash
start-cc-team --stop
start-cc-team --tmux --dir /new/project/path
```

### Q: 任务卡住了怎么办？
A: 按 `Escape` 中断当前任务，然后重新发送：
```bash
tmux send-keys -t cc-team:0.2 Escape
sleep 1
# 重新发送任务
```

### Q: 如何让 CC-Team 读取某个文件？
A: 在任务描述中明确指定文件路径，或用临时文件法：
```bash
tmux send-keys -t cc-team:0.0 "读取 src/main.js 并分析代码结构" Enter
```
