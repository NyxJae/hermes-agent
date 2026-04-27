# 04-消息发送前钩子（transform_messages）需求

> 编号：04
> 创建时间：2026-04-27
> 状态：已对齐，待实现
> 依赖：00-项目基础规范与总体原则

---

## 一、需求概述

当前 Hermes Agent 的插件钩子系统（`hermes_cli/plugins.py`）提供了 `pre_llm_call` 和 `pre_api_request` 两个钩子，但它们都无法**修改即将发送给 LLM 的消息列表**。本需求要求新增一个 `transform_messages` 钩子，在消息完全准备好、即将发送给 API 之前触发，允许插件获取并修改消息列表，然后才最终发送。

**核心原则：**
- 钩子时机：消息列表已完全构建（包含 system prompt、user message、prefill、cache control 等），但尚未调用 API
- 插件能力：可以增删改消息列表中的任意消息
- 影响范围：主 Agent 和子 Agent 都自动支持（钩子放在 `AIAgent.run_conversation` 中）
- 向后兼容：不修改现有钩子语义，纯新增

---

## 二、问题描述

### 2.1 当前行为

现有钩子只能做到：

| 钩子 | 能力 | 局限 |
|------|------|------|
| `pre_llm_call` | 返回文本，注入到当前 turn 的 user message 末尾 | 不能修改消息列表结构（不能插入/删除消息） |
| `pre_api_request` | 只读通知（获取 API 调用信息） | 不能修改任何内容 |

**示例场景（无法实现）：**
- 插件想在 system prompt 后插入一条 `developer` 消息（OpenAI 新角色）
- 插件想删除某条过时的 tool result 消息以节省 token
- 插件想把 user message 拆分成多条消息
- 插件想在消息列表中间插入一条 `user` 消息作为 steering

### 2.2 期望行为

插件可以：
- 获取完整的 `api_messages` 列表（标准 OpenAI 格式）
- 修改后返回新的消息列表
- Hermes 使用插件返回的消息列表继续后续流程（API 调用）

---

## 三、实现方案

### 3.1 钩子定义

在 `hermes_cli/plugins.py` 的 `VALID_HOOKS` 中新增：

```python
VALID_HOOKS: Set[str] = {
    # ... 现有钩子
    "transform_messages",  # 新增
}
```

### 3.2 触发时机

在 `run_agent.py` 的 `run_conversation` 方法中，消息完全构建后、API 调用前：

```python
# 当前位置：约 9782 行（_sanitize_messages_surrogates 之后）
# 插入 transform_messages 钩子

try:
    from hermes_cli.plugins import invoke_hook as _invoke_hook
    _transform_results = _invoke_hook(
        "transform_messages",
        messages=list(api_messages),      # 完整消息列表（含 system prompt）
        session_id=self.session_id or "",
        task_id=effective_task_id,
        model=self.model,
        provider=self.provider,
        api_mode=self.api_mode,
        api_call_count=api_call_count,
        platform=getattr(self, "platform", None) or "",
    )
    # 链式执行：每个插件依次接收上一个插件修改后的消息列表
    for result in _transform_results:
        if result is not None:
            api_messages = result
            logger.debug("transform_messages hook modified api_messages")
except Exception as exc:
    # invoke_hook 内部已处理单个插件异常，这里捕获的是框架级异常
    logger.warning("transform_messages hook failed: %s", exc)
```

### 3.3 消息格式

传递给插件的 `messages` 是标准 OpenAI 格式的消息列表：

```python
[
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "...", "tool_calls": [...]},
    {"role": "tool", "tool_call_id": "...", "content": "..."},
    # ...
]
```

### 3.4 插件返回格式

插件回调函数签名：

```python
def transform_messages_handler(
    messages: List[Dict[str, Any]],
    session_id: str,
    task_id: str,
    model: str,
    provider: str,
    api_mode: str,
    api_call_count: int,
    platform: str,
) -> Optional[List[Dict[str, Any]]]:
    # 修改 messages...
    return modified_messages  # 返回新的消息列表
```

**验证策略：** 与 pi-agent-core 保持一致 —— **无验证**。插件返回的消息列表直接用于后续流程，让 API 调用时的 `_sanitize_api_messages` 和 LLM 提供商自然处理格式问题。

---

## 四、边界情况

| 场景 | 预期行为 |
|------|----------|
| 插件返回空列表 | 视为有效返回，直接传递给后续流程（由 API 调用自然报错） |
| 插件返回格式错误（非 list 或元素非 dict） | 视为有效返回，直接传递给后续流程（由 API 调用自然报错） |
| 插件返回的消息缺少必要字段（如 role） | 不验证，直接传递（由 `_sanitize_api_messages` 或 API 自然处理） |
| 多个插件都返回消息列表 | **链式执行** — 每个插件依次接收上一个插件修改后的消息列表，最终结果 = 所有插件的叠加修改 |
| 插件抛出异常 | 记录 warning，**跳过该插件**，继续执行链中下一个插件（保持链的连续性） |
| 子 Agent（delegate_task） | 自动支持，因为钩子放在 `AIAgent.run_conversation` 中 |

---

## 五、举例说明

### 示例 1：插入一条 developer 消息（OpenAI 新角色）

**插件代码：**

```python
def transform_messages_handler(messages, **kwargs):
    # 在 system prompt 后插入 developer 消息
    new_messages = []
    for msg in messages:
        new_messages.append(msg)
        if msg.get("role") == "system":
            new_messages.append({
                "role": "developer",
                "content": "You must use the provided tools."
            })
    return new_messages
```

**输入消息：**

```python
[
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Hello!"},
]
```

**输出消息：**

```python
[
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "developer", "content": "You must use the provided tools."},
    {"role": "user", "content": "Hello!"},
]
```

---

### 示例 2：删除过时的 tool result 消息

**插件代码：**

```python
def transform_messages_handler(messages, **kwargs):
    # 删除所有 "browser_navigate" 的 tool result（节省 token）
    return [
        msg for msg in messages
        if not (msg.get("role") == "tool" and "browser_navigate" in str(msg.get("content", "")))
    ]
```

**输入消息：**

```python
[
    {"role": "system", "content": "..."},
    {"role": "user", "content": "Search GitHub"},
    {"role": "assistant", "content": "", "tool_calls": [{"id": "1", "function": {"name": "browser_navigate", "arguments": "..."}}]},
    {"role": "tool", "tool_call_id": "1", "content": "Page loaded..."},
    {"role": "assistant", "content": "Found 3 issues."},
]
```

**输出消息：**

```python
[
    {"role": "system", "content": "..."},
    {"role": "user", "content": "Search GitHub"},
    {"role": "assistant", "content": "", "tool_calls": [{"id": "1", "function": {"name": "browser_navigate", "arguments": "..."}}]},
    # tool result 被删除
    {"role": "assistant", "content": "Found 3 issues."},
]
```

---

### 示例 3：子代理超时改进（配合 03-需求）

**场景：** 子代理超时后需要插入一条 system 通知消息，让子代理总结阶段性成果。

**插件代码：**

```python
def transform_messages_handler(messages, api_call_count, **kwargs):
    # 如果这是子代理的第 N 次 API 调用且即将超时
    if api_call_count > 50 and is_subagent():
        # 在最后一条 user/tool 消息后插入超时通知
        new_messages = list(messages)
        new_messages.append({
            "role": "user",
            "content": "【系统通知】你的任务已超时，请不要再调用任何工具，直接总结阶段性成果。"
        })
        return new_messages
    return messages
```

---

## 六、优先级

| 优先级 | 理由 |
|--------|------|
| **P1** | 是 03-子代理超时改进的前置依赖，也支持未来多种插件扩展场景 |

---

## 七、参考实现

参考 pi-mono / pi-agent-core 的 `transformContext` 钩子设计：

```typescript
const agent = new Agent({
  transformContext: async (messages, signal) => {
    // 在 convertToLlm 之前，可以修改消息列表
    messages.push({ role: "user", content: "额外上下文", timestamp: Date.now() });
    return messages;
  },
});
```

---

*本文档定义了 transform_messages 钩子的需求，后续实现需严格遵循。*
