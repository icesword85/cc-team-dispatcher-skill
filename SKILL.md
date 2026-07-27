---
name: cc-team-dispatcher
description: "通过 tmux 调用 CC-Team（4 个不同 AI 模型的 Claude Code 实例）协同工作。当用户需要多模型协作编程、任务分发、架构审查、复杂调试时使用。"
---

# CC-Team Dispatcher Skill

通过 tmux 可靠地向 4 个 Claude Code 实例分发任务，带状态检测、安全发送、接收验证和重试机制。

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

---

## 核心操作流程

### 0. 前置检查（必做）

发送任何指令前，先确认 session 和窗格存在：

```bash
tmux has-session -t cc-team 2>/dev/null && echo "OK" || echo "NO_SESSION"
```

如果 session 不存在，先启动：

```bash
start-cc-team --tmux --dir /path/to/project
```

等待所有窗格初始化完成（约 10~20 秒），再继续。

---

### 1. 检测 Claude Code 是否空闲（发送前必做）

**函数：`cc_is_ready <pane>`**

原理：连续 2 次捕获窗格内容，比较末尾 3 行是否不变 + 末行含提示符。

```bash
cc_is_ready() {
  local pane=$1
  local snap1=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -10 | tail -5)
  sleep 1
  local snap2=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -10 | tail -5)
  # 内容静止 且 末行是提示符
  if [ "$snap1" = "$snap2" ] && echo "$snap2" | tail -1 | grep -qE '[❯>$] *$'; then
    return 0
  else
    return 1
  fi
}
```

等待空闲（超时 60s）：

```bash
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

### 2. 安全发送指令（解决特殊字符问题）

**核心原则：不要用 `send-keys` 直接发长文本。** 使用 tmux 的 buffer 机制 + 粘贴，避免 shell 解析特殊字符（`$`、`` ` ``、`"`、`'`、换行等）。

**函数：`cc_send <pane> <task_text>`**

```bash
cc_send() {
  local pane=$1
  local task=$2

  # 1. 写入临时文件（用 heredoc 保证原样写入）
  local tmpfile
  tmpfile=$(mktemp /tmp/cc-task-XXXXXX.txt)
  cat > "$tmpfile" <<'CC_TASK_EOF'
$task
CC_TASK_EOF
  # 注意：上面 $task 是占位示意，实际调用时把任务文本写到 heredoc 内部

  # 2. 加载到 tmux buffer
  tmux load-buffer -b cc-task-buf "$tmpfile"
  rm -f "$tmpfile"

  # 3. 先确保 command line 是空的（Ctrl+C 清掉可能的残留）
  tmux send-keys -t "cc-team:0.$pane" C-c
  sleep 0.3

  # 4. 粘贴 buffer 内容
  tmux paste-buffer -b cc-task-buf -t "cc-team:0.$pane"

  # 5. 回车发送
  sleep 0.5
  tmux send-keys -t "cc-team:0.$pane" Enter

  # 6. 清理 buffer
  tmux delete-buffer -b cc-task-buf 2>/dev/null
}
```

**实际使用时的完整写法**（把任务放在 heredoc 里，原样保留所有字符）：

```bash
# 示例：向窗格 0 发送一段含特殊字符的任务
TMPFILE=$(mktemp /tmp/cc-task-XXXXXX.txt)
cat > "$TMPFILE" <<'ENDOFTASK'
请帮我重构 src/utils/auth.ts，要求：
1. 把 `verifyToken()` 拆成 3 个函数
2. 错误信息用 "error_${code}" 格式
3. 确保 $TOKEN 环境变量不会被打印
ENDOFTASK

tmux load-buffer -b cc-task-buf "$TMPFILE"
rm -f "$TMPFILE"
tmux send-keys -t cc-team:0.0 C-c
sleep 0.3
tmux paste-buffer -b cc-task-buf -t cc-team:0.0
sleep 0.5
tmux send-keys -t cc-team:0.0 Enter
tmux delete-buffer -b cc-task-buf 2>/dev/null
```

> **为什么不用 `send-keys` 直接发？**
> - `send-keys` 会解释特殊键名（`Enter`、`Escape`、`C-c` 等）
> - 长文本中如果含 `$`、`` ` ``，会被目标窗格的 shell 展开
> - 超过 ~1KB 的文本可能丢字符
> - buffer + paste 是 tmux 官方推荐的长文本发送方式

---

### 3. 验证指令是否被成功接收

发送后 2~3 秒，检查窗格末尾是否出现了任务的开头关键词。

**函数：`cc_verify_received <pane> <keyword>`**

```bash
cc_verify_received() {
  local pane=$1
  local keyword=$2
  sleep 3
  local tail
  tail=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -20)
  # 检查最近 20 行是否包含任务关键词（说明指令已上屏）
  if echo "$tail" | grep -qF "$keyword"; then
    return 0
  else
    return 1
  fi
}
```

验证通过标准：
- 任务文本的前 20 个字符出现在窗格末尾
- 出现 Claude Code 的思考/响应前缀（如 `Thinking...` 或工具调用）

---

### 4. 完整发送流程（含重试）

**函数：`cc_dispatch <pane> <task_text> <verify_keyword> [max_retries=3]`**

```
流程：
1. 等待窗格空闲（超时 60s）→ 失败则报错
2. 用 buffer 方式发送任务
3. 等待 3s 后验证是否收到
4. 未收到 → Ctrl+C 清场 → 重试（最多 3 次）
5. 全部重试失败 → 返回错误
```

bash 实现：

```bash
cc_dispatch() {
  local pane=$1
  local task_file=$2      # 已写好任务的文件路径
  local keyword=$3        # 任务中独特的开头关键词，用于验证
  local max_retries=${4:-3}

  # 等待空闲
  echo "[pane $pane] 等待空闲..."
  if ! cc_wait_ready "$pane"; then
    echo "[pane $pane] ERROR: 等待空闲超时"
    return 1
  fi
  echo "[pane $pane] 空闲，准备发送"

  local attempt=1
  while [ $attempt -le $max_retries ]; do
    echo "[pane $pane] 第 $attempt 次尝试发送..."

    # 发送
    tmux load-buffer -b cc-task-buf "$task_file"
    tmux send-keys -t "cc-team:0.$pane" C-c
    sleep 0.3
    tmux paste-buffer -b cc-task-buf -t "cc-team:0.$pane"
    sleep 0.5
    tmux send-keys -t "cc-team:0.$pane" Enter
    tmux delete-buffer -b cc-task-buf 2>/dev/null

    # 验证
    sleep 3
    local tail
    tail=$(tmux capture-pane -t "cc-team:0.$pane" -p -S -20)
    if echo "$tail" | grep -qF "$keyword"; then
      echo "[pane $pane] ✅ 指令发送成功"
      return 0
    fi

    echo "[pane $pane] 第 $attempt 次发送未验证到，重试..."
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

# 等待完成：轮询直到连续 5 秒无新输出且回到提示符
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
      # 内容 5 秒没变，检查是否回到提示符
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

## 最佳实践

1. **始终用 buffer 发送**，不要用 `send-keys` 发超过一行的任务
2. **发送前等空闲**，这是最容易被忽略但最关键的一步
3. **关键词验证**用任务第一句最独特的词（避免匹配历史内容）
4. **多窗格并发**时，每个窗格独立重试，不互相阻塞
5. **长任务**定期用 `capture-pane` 检查进度，不要傻等
6. **任务文本含中文**没问题，tmux buffer 支持 UTF-8
7. **幂等性**：设计任务时尽量让它可重复执行，方便重试

## 常见故障排查

| 现象 | 可能原因 | 解决 |
|------|---------|------|
| 指令没反应 | CC 还在思考，指令插到中间了 | 发前等空闲 |
| 只有部分文本 | send-keys 超长丢字符 | 改用 buffer |
| 出现奇怪输出 | 特殊字符被 shell 解析 | 改用 buffer + heredoc |
| 验证总失败 | keyword 太普通或任务没上屏 | 换独特关键词，增加等待时间 |
| session 不存在 | CC-Team 没启动或崩了 | `start-cc-team --tmux` 重启 |
